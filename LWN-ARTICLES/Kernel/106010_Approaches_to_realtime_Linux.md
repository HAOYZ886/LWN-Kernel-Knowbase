---
title: Approaches to realtime Linux
url: https://lwn.net/Articles/106010/
date: "October 12, 2004"
category: Interrupts; Latency; Preemption; Realtime; Voluntary preemption
author: 
---

Using Linux systems for realtime tasks has long been an area of interest. In the last couple of weeks, a number of projects working to implement realtime response have posted their work. This article looks at the patches posted recently to get a sense for where the realtime projects are headed. 

### The realtime LSM

A relatively simple contribution is the [realtime security module](<https://lwn.net/Articles/106009/>) by Torben Hohn and Jack O'Quin. This module does not actually add any new realtime features to the kernel; instead, it uses the LSM hooks to let users belonging to a specific group use more of the system's resources. In particular, it adds the `CAP_SYS_NICE`, `CAP_IPC_LOCK`, and `CAP_SYS_RESOURCE` capabilities to the selected group. These capabilities allow the affected processes to raise their priority, lock memory into RAM, and generally to exceed resource limits. Granting capabilities in this way goes somewhat beyond the usual "restrictive hooks only" practice for security modules, but there have not been any complaints on that score. 

### MontaVista's patch

The event which really stirred up the discussion, however, was the posting of the [realtime kernel patch set](<https://lwn.net/Articles/105866/>) by MontaVista's Sven-Thorsten Dietrich. This highly intrusive patch attempts to minimize system response latency by taking the preemptible kernel approach to its limit. In comparison, the current preemption approach, which is considered to be too risky to use by most distributors, is a half measure at best. 

MontaVista's patch begins by adopting [the "IRQ threads" patch](<https://lwn.net/Articles/95334/>) posted by Ingo Molnar. This patch moves the running of most interrupt handlers into a separate kernel thread which competes with the others for processor time. Once that is done, interrupt handlers become preemptible and are far less likely to stall the system for long periods of time. 

The biggest source of latency in the kernel then becomes critical sections protected by spinlocks. So why not make those sections preemptible as well? To that end, the [PMutex patch](<http://inf3-www.informatik.unibw-muenchen.de/research/linux/mutex/>) has been adapted to the 2.6 kernel. This patch implements blocking mutexes, similar to the existing kernel semaphores. The PMutex version, however, has a simple priority inheritance mechanism; processes holding a mutex can have their priority bumped up temporarily so that they get their work done and release the mutex as quickly as possible. Among other things, this approach helps to minimize priority inversion problems. 

The biggest change is replacing of most spinlocks in the system with the new mutexes; the patch uses a set of preprocessor macros to turn `spinlock_t`, and the operations on spinlocks, into their mutex equivalents. In one step, most critical sections become preemptible and no longer are part of the latency problem. As an added bonus, the moving of interrupt handlers to their own thread means that interrupt handlers can no longer deadlock with non-interrupt code when contending for the same lock; that means that it is no longer necessary to disable interrupts when taking a lock which might also be used by an interrupt handler. 

There are, of course, a few nagging little problems to deal with. Some code in the system really _shouldn't_ be preempted while holding a lock. In particular, code which might be in the middle of programming hardware registers, the page table handling code, and the scheduler itself need to be allowed to do their job in peace. It is hard, after all, to imagine a scenario where preempting the scheduler will lead to good things. So a number of places in the kernel cannot be switched from spinlocks to the new mutexes. 

The realtime patch attempts to handle these cases by creating a new `_spinlock_t` type, which is just the old `spinlock_t` under a newer, uglier name. The spinlock primitives have been renamed in the same way (e.g. `_spin_lock()`). Code which truly needs an old-style spinlock is then hacked up to use the new names, and it functions as before. Except for some files, where the developers were able to include `<linux/spin_undefs.h>`, which restores the old functionality under the old names. The header file rightly describes this technique as "a dirty, dirty hack." But it does make the patch smaller. 

Needless to say, the task of sifting through every lock in the kernel to figure out which ones cannot be changed to mutexes is a long and error-prone process. In fact, the job is nowhere near complete, and the MontaVista patch is, by its authors' admission, marginally stable on uniprocessor systems, unstable on SMP systems, and unrunnable on hyperthreaded systems. But you have to start somewhere. 

### Ingo's fully preemptible kernel

Ingo Molnar liked that start, but had some issues with it. So he went off for two days and [created a better version](<https://lwn.net/Articles/105948/>), which has been folded into his "voluntary preemption" series of patches. Ingo takes the same basic approach used by the MontaVista patch, but with some changes: 

  * The PMutex patch is not used; instead, Ingo uses the existing kernel semaphore implementation. His argument is that semaphores work on all architectures, while PMutexes currently only work on x86. It would be better to hack priority inheritance into the existing semaphores, and thus make it available to all of the current semaphore users as well as those converted over from spinlocks. Ingo's patch does not currently implement priority inheritance, however. 

  * Through some preprocessor trickery, Ingo was able to avoid changing all of the spinlock calls. Preserving "old style" spinlock behavior is simply a matter of changing the type of the lock to `raw_spinlock_t` and, perhaps, changing the initialization of the lock. The actual `spin_lock()` and related calls do the right thing with either a "raw" spinlock or a new semaphore-based mutex. Think of it as a sort of poor man's polymorphic lock type. 

  * Ingo found a much larger set of core locks which must use the true spinlock type. This was done partly through a set of checks built into the kernel which complain when the wrong type of lock is being used. With Ingo's patch, some 90 spinlocks remain in the kernel (in comparison, MontaVista preserved about 30 of them). Even so, thanks to the reworked locking primitives, Ingo's patch is much smaller than the MontaVista patch. 

Ingo would like to reduce the number of remaining spinlocks, but he warns that a number of "core infrastructure" changes will be required first. In particular, code using [read-copy-update](<https://lwn.net/Articles/37889/>) must continue to use spinlocks for now; allowing code which holds a reference to an RCU-protected structure to be preempted would break one of the core RCU assumptions. MontaVista has apparently taken a stab at the RCU issue, but does not yet have a patch which they are ready to circulate. 

Ingo continues to post patches at a furious rate; things are evolving quickly on this front. 

### RTAI/Fusion

Meanwhile, the _real_ realtime people point out that none of this work provides deterministic, quantifiable latencies. It does help to reduce latency, but it cannot provide guarantees. A "realtime" system without latency guarantees may be suitable for a number of tasks, but it still isn't up to the challenge of running a nuclear power plant, an airliner's flight management system, or an extra-fast IRC spambot. If it absolutely, positively must respond within a few microseconds, you need a real realtime system. 

There are two longstanding Linux projects which are intended to provide this sort of deterministic response: [RTLinux](<http://www.fsmlabs.com/products/openrtlinux/>) and [RTAI](<http://www.aero.polimi.it/~rtai/>). There is the obligatory bad blood between the two, complicated by a software patent held by the RTLinux camp. 

The RTLinux approach (and the subject of the patent) is to put the hardware under the control of a small, hard realtime system, and to run the whole of Linux as a single, low-priority task under the realtime system. Access to the realtime mode is obtained by writing a kernel module which uses a highly restricted set of primitives. Channels have been provided for communicating between the realtime module and the normal Linux user space. Since the realtime side of the system controls the hardware and gets first claim on its resources, it is possible to guarantee a maximum response time. 

RTAI initially used that approach, but has since shifted to running under the [Adeos kernel](<http://www.gna.org/projects/adeos/>). Adeos is essentially a "hyperviser" system which runs _both_ Linux and a real-time system as subsidiary tasks, and allows the two to communicate. It allows a pecking order to be established between the secondary operating systems so that the realtime component can respond first to hardware events. This approach is said to be more flexible and also to avoid the RTLinux patent. Working with RTAI still requires writing kernel-mode code to handle the hard realtime part of the task. 

In response to the current discussion, Philippe Gerum surfaced with [an introduction to the RTAI/Fusion project](<https://lwn.net/Articles/106016/>). This project, which is "a branch" of the RTAI effort, is looking for a middle ground between the low-latency efforts and the full RTAI mode of operation; its goal is to allow code to be written for the Linux user space, with access to regular Linux facilities, but still being able to provide deterministic, bounded response times. To this end, RTAI/Fusion provides two operating modes for realtime tasks: 

  * The "hardened" mode offers strict latency guarantees, but programs must restrict themselves to the services provided by RTAI. A subset of Linux system calls are available as RTAI services, but most of them are not. 

  * When a task invokes a system call which cannot be implemented in the hardened mode, it is shifted over to the secondary ("shielded") scheduling mode. This mode is similar to the realtime modes implemented by MontaVista and Ingo Molnar; all Linux services are available, but the maximum latency may be higher. The RTAI/Fusion shielded mode defers most interrupt processing while the realtime task is running, which is said to improve latency somewhat. 

Processes may move between the two modes at will. 

The end result is a blurring of the line between regular Linux processes and the hard realtime variety. Developers can select the mode which best suits their needs while running under the same system, and they can use different modes for different phases of a program's execution. RTAI/Fusion might yet succeed in the task of combining a general-purpose operating system with hard realtime operation. 

### In conclusion...

Whether any of the work described here will make it into the mainline kernel is another question. The preemptible kernel patch, which was far less ambitious, has still not been accepted by many developers. Removing most spinlocks and making the kernel fully preemptible will certainly be an even harder sell. It is an intrusive change which could take some time to stabilize fully. If a fully-preemptible, closer-to-realtime kernel does pass muster with the kernel developers, it may well be the sort of development that finally forces the creation of a 2.7 branch. 

Another challenge will be building a consensus around the idea that the mainline kernel should even try to be suitable for hard realtime tasks. The kernel developers are, as a rule, opposed to changes which benefit a tiny minority of users, but which impose costs on all users. Merging intrusive patches for the sake of realtime response looks like that sort of change to many. Before mainline Linux can truly claim to be a realtime system, the relevant patches will have to prove themselves to be highly stable and without penalty for "regular" users.
