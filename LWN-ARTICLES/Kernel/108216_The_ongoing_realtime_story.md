---
title: The ongoing realtime story
url: https://lwn.net/Articles/108216/
date: "October 27, 2004"
category: Realtime
author: 
---

The efforts to bring hard realtime response to Linux continue. For those of you following along at home, here is a summary of the latest realtime Linux developments. 

Ingo Molnar continues to crank out patches at a high rate. The latest, ~~[-RT-2.6.9-mm1-V0](<https://lwn.net/Articles/108008/>)~~ [-RT-2.6.9-mm1-V0.3](<https://lwn.net/Articles/108449/>), is advertised as being rather more experimental than its predecessors - which is saying something. This patch set brings preemptible mutexes to (almost) the last, most difficult parts of the kernel, including the low-level memory allocators, wait queue code, kernel timers, and more. Says Ingo: 

this is probably the last 'big leap forward' in terms of the scope of the patch. (having reached the ultimate scope: it now encompasses everything ;) 

Some small pieces of this work have been put forward as independent patches; these include the enhancements to the completion interface mentioned [last week](<https://lwn.net/Articles/106560/>). Linus has also made a couple of changes to the big kernel lock code in support of this sort of work: the BKL functions are now entirely out-of-line, and some of the code for managing the BKL itself has been made preemptible. 

Ingo's patch also changes a number of semaphores in the kernel over to completions. For situations where one kernel thread needs to notify another that some task has been finished, completions are a better interface: they make the intent of the code clear, and they are better optimized for that use. Some of those patches have been posted separately as well, leading to some pushback from kernel developers who believe that their use of semaphores for that purpose is entirely legitimate. Bill Huey, the developer behind the mmlinux realtime project, is the person who has been pushing some of those patches; he [responded](<https://lwn.net/Articles/108221/>) to the resistance in this way: 

Well, this is something that's got to be considered by the larger Linux community and whether these conventions are to be kept or removed. It's a larger issue than what can be address in Ingo's preemption patch, but with inevitable need for something like this in the kernel (hard RT) it's really unavoidable collision. IMO, it's got to go, which is a nasty change. 

This, of course, is just the sort of talk which will put many kernel developers off the realtime patches entirely; some of Mr. Huey's subsequent postings, being rather more inflammatory, did not help the situation either. Ingo [went into damage control mode](<https://lwn.net/Articles/108223/>) and smoothed things over, for now. If and when the realtime preemption patch is put forward for inclusion, however, chances are that the discussion could get heated indeed. 

Paul McKenney, meanwhile, [expressed a discomfort](<https://lwn.net/Articles/107930/>) with the realtime work which must certainly be felt by many: 

The problem is that the entire OS kernel must be modified to ensure that all code paths are deterministic. It would be much better if there was an evolutionary path to hard realtime. 

His message included a patch intended to point toward such a path. This patch, which assumes an SMP system, works by setting aside one CPU as a purely realtime processor; it is not part of the regular scheduling mechanism. Realtime processes may be assigned to that CPU by the system administrator. If they mostly work in user mode, all is well; they have a dedicated processor and need not worry about latency. As soon as a realtime process invokes a system call, however, it goes into non-deterministic mode and is booted out to one of the system's other processors. In this way, the dedicated, real-time processor never gets stuck waiting for a lock. 

The downside, of course, is that, every now and then, it is actually nice to be able to use system calls. Paul's idea was that each Linux system call could be examined individually, and, if warranted, modified to be entirely preemptible. When any particular system call reaches a state where it is considered to be deterministic, it could be added to a list of such calls, and realtime processes using it need not be shifted away from the realtime processor. Over time, this list would grow to the point that realtime tasks which do actual, interesting work could be run on the mainline Linux kernel. In the mean time, there would be no need for a major flag day where the entire kernel locking scheme is changed at once. 

The real challenge with this approach would be to make I/O deterministic, since realtime processes usually must act in response to external events. That cannot be done until it is clear that all filesystems and device drivers have been made entirely preemptible - and, at that point, much of the system has been affected. Meanwhile, it turns out that the 2.6.9 kernel already has part of this mechanism: the new `isolcpus=` boot parameter excludes one or more processors from regular scheduling. The scheme for migrating realtime processes when they invoke a non-deterministic system call is not present, however.
