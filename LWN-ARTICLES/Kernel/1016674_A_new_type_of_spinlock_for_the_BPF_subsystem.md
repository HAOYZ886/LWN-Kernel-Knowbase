---
title: A new type of spinlock for the BPF subsystem
url: https://lwn.net/Articles/1016674/
date: "April 9, 2025"
category: "BPF-Concurrency"
author: "By Daroc Alden April 9, 2025 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Daroc Alden**  
April 9, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The 6.15 merge window saw the inclusion of a new type of lock for BPF programs: a resilient queued spinlock that Kumar Kartikeya Dwivedi has been working on for some time. Eventually, he hopes to convert all of the spinlocks currently used in the BPF subsystem to his new lock. He gave a remote presentation about the design of the lock at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF summit. 

Dwivedi began by providing a bit of background on existing locking in BPF. In 2019, Alexei Starovoitov [ introduced](<https://lwn.net/ml/netdev/20190116050830.1881316-1-ast@kernel.org/>) `bpf_spin_lock()`, which allowed BPF programs to update map values atomically. But the lock came with the serious limitation that a BPF program could only hold one lock at a time, and could not perform any function calls while the lock was held. This let the verifier ensure that BPF programs could not deadlock, but was awkward to use, Dwivedi said. 

In 2022, [ sched_ext](<https://lwn.net/Articles/922405/>) led to the introduction of more kernel data structures to BPF, including linked lists and red-black trees. The verifier was tasked with ensuring that the BPF program could lock and unlock those data structures correctly while manipulating them, but still only supported holding one lock at a time, and only allowed restricted operations while it was held. Some algorithms are much easier to express if the program is allowed to take two locks, Dwivedi explained. So this was a lot of friction to impose on BPF users, all for the sake of avoiding deadlocks. 

The thing is, the syzbot kernel fuzzing system regularly finds deadlocks in the BPF runtime. The verifier cannot help with those, either, since they're occurring in kernel code and not in the loaded BPF programs. This is ""an endless source of issues"" that the BPF developers need to find some way to deal with, he said. 

So the problem that Dwivedi set for himself was: how to guarantee forward progress in the kernel, when applying more static analysis to the kernel is infeasible? Furthermore, he wanted to find a solution that was scalable, and that could isolate faults to the specific BPF program that introduced the problem, without causing other programs to suffer. This is clearly a tall order, but Dwivedi believes that he has a preliminary solution. 

#### Resilient queued spinlocks

His solution is to introduce a new kind of lock — a resilient queued spinlock — that will dynamically check for deadlocks (and livelocks) at run time, and encourage its use in areas of the kernel that frequently suffer from locking problems. The basic structure of the lock is (for most kernel configurations) four bytes: 

  * `locked` — a one-byte value that is either one, when the lock is held, or zero, when it's free.
  * `pending` — a one-byte value that is used to indicate that a second thread is waiting on the lock.
  * `tail` — a two-byte index into a table of wait queues that is used when more than two threads want the lock.

When a thread is waiting on the lock (and would otherwise be wasting CPU time spinning), it checks a per-CPU table of held locks against information stored in the lock's queue in order to detect deadlocks. If waiting for the lock takes too long without the lock becoming free, an error is also returned indicating a livelock. 

Dwivedi went through what the process of locking and unlocking the lock looks like at different levels of contention. When the lock is uncontended, it works exactly like the kernel's current spinlocks: the thread that wants to acquire the lock uses a compare-and-exchange instruction to set `locked` to one. When it's done, it sets it back to zero. When two threads want the lock, the sequence is similar, except that the second thread will set `pending` to indicate that it is waiting and start spinning on the lock. 

When more than two threads want the lock, things get interesting. The third thread to try to acquire the lock sees that the `pending` bit is set, adds itself to a new queue, and then points the `tail` field of the lock to the entry for that queue. Then, as the head of the queue, it becomes responsible for running checks for deadlocks. Any new threads that attempt to acquire the lock add themselves to the queue. 

Eventually, the thread holding the lock will give it up, prompting the pending thread to grab it and reset `pending` to zero. Because the thread at the head of the queue was handling deadlock checks, the pending thread was free to spin on the lock, so the latency from the lock being freed until the pending thread can acquire it should be low. If the thread holding the lock doesn't give it up in a reasonable amount of time, the thread at the head of the queue also notices that, and the waiting threads all remove themselves from the queue and return error messages. 

The design of the lock is intended to have performance competitive with the kernel's existing spinlocks, while still letting deadlocks be detected. To support this, Dwivedi showed a number of graphs from benchmarks such as locktorture and will-it-scale, which are also available in [ his commit message](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6ffb9017e932>). In short, the new lock looks as though it performs only slightly worse than the existing spinlock on an Intel x86_64 CPU, and pretty much identically on an arm64 CPU — although there was the normal amount of noise in his performance data making the difference somewhat hard to detect. 

#### Next steps

Now that the lock is available in the kernel (although it wasn't at the time of Dwivedi's presentation), he has plans for how to make use of it. The place he most wants to see the lock used is in the BPF runtime, both to help eliminate the existing deadlock problems there, and because that will let the BPF developers relax some of the restrictions on BPF programs that hold locks. He also wants to work on reporting detected deadlocks to user space, which is currently not done. 

He pointed the audience at [ his and Starovoitov's white paper](<https://github.com/kkdwivedi/rqspinlock/blob/main/rqspinlock.pdf>) for anyone interested in additional details of how the deadlock-detection algorithm works. I asked how long he expected it to take to adapt the whole BPF runtime to use his new lock; Dwivedi said that the work was not complicated, he just wanted to avoid doing it in the initial patch set. ""Once the whole thing lands in mainline, we will want to convert more parts."" 

Starovoitov asked about his ideas for replacing the existing spinlocks. Specifically, the existing `bpf_spin_lock()` function returns `void`, and has no possibility of failure, which means adding error paths to use the new lock. Dwivedi said that there was some additional needed work to allow multiple locks to be held at once, but that existing BPF programs should just get canceled if they cause an error. Part of the point of his work is that misbehaving programs should get kicked out of the kernel. In response to some clarifying questions, he said that the goal was that ""the kernel won't break; your program might break"", and that any new BPF programs going forward could check the return value of the lock explicitly if they needed to. 

Starovoitov expressed hope that the new lock would eventually displace all uses of existing spinlocks in the BPF subsystem. Anton Protopopov was concerned about the performance impacts from the changeover, and was unconvinced by Dwivedi's measurements, although the other BPF developers did not seem overly concerned. 

I also asked whether this was the only kind of lock Dwivedi thought the BPF runtime would need. He thought this was the only one needed for now, although he acknowledged that some people have occasionally wanted mutexes. 

With the lock now accepted into the mainline kernel, it seems likely that Dwivedi will be pushing forward with his quest to adapt the BPF subsystem. If that conversion does actually successfully eliminate deadlocks without an undue performance impact, perhaps other areas of the kernel will also find the new lock useful.
