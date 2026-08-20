---
title: "Timer IDs, CRIU, and ABI challenges"
url: https://lwn.net/Articles/1012490/
date: "March 6, 2025"
category: "Checkpointing; Development model-User-space ABI; Releases-6.15; System calls-timer create"
author: "By Jonathan Corbet March 6, 2025"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 6, 2025

The kernel project has usually been willing to make fundamental internal changes if they lead to a better kernel in the end. The project also, though, goes out of its way to avoid breaking interfaces that have been exposed to user space, even if programs come to rely on behavior that was never documented. Sometimes, those two principles come into conflict, leading to a situation where fixing problems within the kernel is either difficult or impossible. This sort of situation has been impeding performance improvements in the kernel's POSIX timers implementation for some time, but it appears that a solution has been found. 

#### Timers and CRIU

The POSIX timers API allows a process to create its own private interval timers based on any of the clocks provided by the kernel. A process calls [`timer_create()`](<https://man7.org/linux/man-pages/man2/timer_create.2.html>) to create such a timer: 

```
int timer_create(clockid_t clockid, struct sigevent *sevp, timer_t *id);
```

The `id` argument is a pointer to a location where the kernel can return an ID used to identify the new timer; it is of type `timer_t`, which maps eventually to an `int`. Various other system calls can use that ID to arm or disarm the timer, query its current status, or delete it entirely. The man page for `timer_create()` indicates that each created timer will have an ID that is unique within the creating process, but makes no other promises about the returned value. 

The "unique within the process" guarantee came with the 3.10 kernel release in 2013; previously, the timer IDs were unique system-wide. To understand that change, one has to look at the [Checkpoint/Restore in Userspace (CRIU)](<https://criu.org/Main_Page>) project, which has long worked on the ability to save the state of a group of processes to persistent storage, then restore that group at a future time, possibly on a different system. Reconstructing the state of a set of processes well enough that the processes themselves are not aware of having been restored in this way is a challenging task; the CRIU developers have often struggled to get all of the pieces working (and to keep them that way). 

POSIX timers were one of the places where they ran into trouble. To restore a process that is using timers, CRIU must be able to recreate the timers with the same ID they had at checkpoint time, but the system-call API provides no way to request a specific timer ID. Even if such an ability existed, though, the existence of a single, system-wide ID space for timers was an insurmountable problem; CRIU might try to recreate a timer for a process, only to find that some other, unrelated process in the system already had a timer with that ID. In such cases, the restore would fail. 

This problem was addressed with [this patch](<https://git.kernel.org/linus/5ed67f05f66c>) from Pavel Emelyanov, which implemented a new hash table to store timer IDs. That table was still global, but the timer IDs kept therein took the identity of the owning process (specifically, the address of its [`signal_struct`](<https://elixir.bootlin.com/linux/v6.13.4/source/include/linux/sched/signal.h#L87>) structure) into account, separating each process's timer IDs from all the others. At that point, the problem of ID collision when restoring a process went away. 

The other problem — the lack of a way to request a specific timer ID — remained, though. To address that problem, CRIU stuck with the approach it had used before, which was based on some internal knowledge about how the kernel allocates those IDs. There is a simple, per-process counter, starting at zero, that is used for timer IDs; that counter is incremented every time a new timer is created. So a series of `timer_create()` calls will yield a predictable series of IDs, counting through the integer space. When CRIU must create a timer with a specific ID within a to-be-restored process, it takes advantage of this knowledge and simply runs a loop, allocating and destroying timers, until the requested ID is returned. 

If a process only creates a small number of timers in its lifetime, this linear ID search will not take long. Checkpointing, though, is often used on long-running processes in order to save their state should something go wrong partway through. That kind of process, if it regularly creates and destroys timers, can end up with IDs that are spread out widely in the integer space. That, in turn, means it can take a long time to land on the needed ID at restore time. 

#### Without a paddle

In 2023, Thomas Gleixner sent [this summary](<https://lwn.net/ml/all/87ednpyyeo.ffs@tglx/>) in response to a timer bug report; he noted that in some cases, the allocation loop ""will run for at least an hour to restore a single timer"". That is not the speedy restore operation that CRIU users may have been hoping for. But the real problem at the time was that the requirement to allocate timer IDs sequentially in the kernel was getting in the way of some needed changes to the internal global hash table, which, in turn, were blocking other changes within the timer subsystem. Since that behavior could not be changed without breaking CRIU, Gleixner concluded that the kernel was ""up the regression creek without a paddle"". 

At the time, some possible solutions were considered. Reducing the ID space from `0..INT_MAX` to something smaller could speed the ID search, but it would still be an ABI break; CRIU would no longer be able to restore any process that had created timers with a larger ID. A new system call to create a timer with a given ID was another possibility but, due to how the timer API works (and the `sigevent` structure it accepts), the 64-bit and 32-bit versions of the system call could not be made compatible. That would require the addition of another "compat" system call, which is something the kernel developers have gone out of their way to avoid for some time. In the end, the conversation wound down with no solution being found. 

In mid-February 2025, networking developer Eric Dumazet posted [a patch series](<https://lwn.net/ml/all/20250219125522.2535263-1-edumazet@google.com/>) aimed at reducing locking contention in the kernel's timer code, citing ""incidents involving gazillions of concurrent posix timers"". That work elicited some testy responses from Gleixner, but there was no questioning the existence of a real problem. So Gleixner went off to create [his own patch series](<https://lwn.net/ml/all/20250302185753.311903554@linutronix.de>), incorporating Dumazet's work, but then aiming to solve the other problems as well. Most of the series is focused on implementing a new hash table that lacks the performance problems found in current kernel; benchmark results included in the cover letter show that some success was achieved on that front. 

#### A better solution for CRIU

But then Gleixner set out to solve the CRIU problem as well. Rather than create a new system call to enable the creation of a timer with a specific ID, though, he concluded that the `id` argument to `timer_create()` could be used to provide that ID. All that is needed is a flag to tell `timer_create()` to use the provided value rather than generating its own ... but `timer_create()` has no `flags` argument. So, if `timer_create()` is to gain the ability to read a timer ID from user space, some other way needs to be found to let it know that this behavior is requested. 

The answer is a pair of new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) operations. A call like this: 

```
prctl(PR_TIMER_CREATE_RESTORE_IDS, PR_TIMER_CREATE_RESTORE_IDS_ON);
```

will cause the calling process to enter a "timer restoration mode" that causes `timer_create()` to read the requested timer ID from the location pointed to by the `id` parameter passed from user space. The special value `TIMER_ANY_ID` can provided in cases where user space does not have an ID it would like to request. Another `prctl()` call with `PR_TIMER_CREATE_RESTORE_IDS_OFF` will exit the restoration mode, causing any subsequent `timer_create()` calls to generate an ID internally as usual. 

This functionality is narrowly aimed at CRIU's needs. Normally, adding this kind of process-wide state would be an invitation for problems; some distant thread could make a `timer_create()` call while the restoration mode is enabled, but expecting the old behavior, and thus be unpleasantly surprised. But CRIU can use this mode at the special point where the restarted processes have been created, but are not yet allowed to resume running at the spot where they were checkpointed. At that time, CRIU is entirely in control and can manage the state properly. 

Another important point is that the `prctl()` call will fail on an older kernel that does not support the timer restoration mode. When CRIU sees that failure, it can go back to the old, brute-force method of allocating timers. The CRIU developers will thus be able to take advantage of the new API while maintaining compatibility for users on older kernels. 

One problem that will remain even after this series is merged is that the sequential-allocation behavior of `timer_create()`, in the absence of the new `prctl()` operation, is still part of the kernel's ABI. The timer developers never meant to make that promise, but they are stuck with it for as long as CRIU installations continue to depend on it. The good news is that updating CRIU will generally be necessary for users who update their kernels anyway, since that is the only way to get support for newer kernel features. So, perhaps before too long, the sequential-allocation guarantee for `timer_create()` can be retired — unless some other user that depends on it emerges from the woodwork.
