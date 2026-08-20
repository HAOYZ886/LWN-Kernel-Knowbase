---
title: Dropping the timer tick — for real this time
url: https://lwn.net/Articles/659490/
date: "October 7, 2015"
category: Dynamic tick
author: "By Jonathan Corbet October 7, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 7, 2015

The kernel reached a landmark of sorts in 2013 when [nearly full tickless support](<https://lwn.net/Articles/549580/>) was merged for the 3.10 release. This work allows the timer interrupt to be disabled when a single process is running on a CPU; that gives the process uninterrupted access to the CPU for as long as it avoids calling into the kernel. It turns out, though, that some applications have such stringent requirements that even "nearly full" support is not good enough. For those, Chris Metcalf's [task isolated mode](<https://lwn.net/Articles/658596/>) may fill the bill — but first, it must get past the review process. 

Tickless mode is generally needed by applications that cannot afford to ever lose access to the CPU. A common example is high-bandwidth networking applications that do all of their packet processing (and protocol work) in user space. If this type of application is interrupted, it can drop packets into an unsightly mess on the floor. Tickless mode seeks to prevent such troubles by directing all interrupts and kernel housekeeping work to other CPUs so that the application never stops running as long as it doesn't, itself, call into the kernel. 

The current tickless mode is not a 100% solution, though, in that it still allows the timer interrupt to fire once every second. Any other timers that may have been set (perhaps in response to something the application did) will also be allowed to fire. For some applications, a reduction in interrupts by two or three orders of magnitude is still not enough; they truly want it all, even if the cost is high. 

Chris's patch is meant to let these applications have it all. An application running on a CPU that has been configured for tickless operation (done with the `nohz_full=` boot option) can invoke a new `prctl()` command (`PR_SET_TASK_ISOLATION`) to enter the fully isolated mode. Processes that want to ensure that they will stay in this mode can turn on the strict enforcement option by adding the `PR_TASK_ISOLATION_STRICT` bit in the `prctl()` call; should a process do anything that causes an entry into the kernel while strict mode is on, it will be summarily killed. 

The isolation mode is much like the ordinary tickless mode, with one exception: the kernel carries out a number of actions whenever that process returns to user space to ensure that it will not be interrupted. That return can happen at the end of the `prctl()` enabling isolation mode, or, if the strict option has not been set, after any arbitrary system call while isolation mode is enabled. 

Some of the return-to-user-mode work is fairly straightforward. The memory-management subsystem does some statistics collection through the "vmstat" mechanism, which is run from a delayed workqueue entry; in full isolation mode, this work is turned off. A call to `lru_add_drain()` is made to ensure that the CPU will not be asked to free up CPU-local pages while it is running in user space. And, most controversially, the CPU will busy-wait until there are no more pending timers to run. 

The problem with the busy wait, of course, is that there is no way of knowing just how long it will take for the pending timers to expire. If some bit of code has set a timer for sometime next year, the loop will spin for that long. The code is safe (if inelegant) if one "knows" that, for a given workload, no overly inconvenient timers will be set. But in the wider world that the kernel runs in, assuming that timer users will never get in the way is asking for trouble. For this reason, reviewers like Thomas Gleixner are [insisting](<https://lwn.net/Articles/659500/>) that the timer problem be solved properly, by ensuring that unwanted timers will not be set in the first place. 

Interestingly, both Chris and Thomas seem to agree that it would be best to just not have timers running while isolation mode is active. The difference seems to be that Chris worries that it will never be possible to identify every situation where a timer might be set or to prevent the introduction of new timers in the future. As [he put it](<https://lwn.net/Articles/659502/>): 

In general, the hard task-isolation requirement is something that is of particular interest only to a subset of the kernel community. As the kernel grows, adds features, re-implements functionality, etc., it seems entirely likely that odd bits of deferred functionality might be added in the same way that RCU, workqueues, etc., have done in the past. Or, applications might exercise unusual corners of the kernel's semantics and come across an existing mechanism that ends up enabling kernel ticks (maybe only one or two) before returning to userspace. The proposed busy-loop just prevents that from damaging the application. 

This mode, he added, is only active for applications that have explicitly requested it, and only when they are the only process running on a core that has been configured for tickless operation. In such situations, the busy wait for timer events should not cause any harm. 

Thomas, though, is convinced that the current approach is dancing around a couple of problems that have to be solved in the general case. There are reasons for the continued existence of the once-per-second tick; they include CPU-usage accounting and more. Just disabling that tick without getting a handle on those problems may work for Chris's use case, but it's likely to run into trouble with others. Until these problems are solved, getting this work into the kernel is likely to be difficult. 

Chris seems prepared to try to solve these problems. He has [proposed reworking the patch set](<https://lwn.net/Articles/659509/>) to eliminate the busy-wait loop; instead, it would simply reschedule if any other processes are ready to run. The presence of other runnable processes is the main reason for an inability to disable the timer tick; it rules out running in the isolated mode in any case. Then, he plans to add a debugging-oriented mode that disables the once-per-second tick in "tickless" mode, making it possible to research the problems caused by doing without a timer tick entirely. With that infrastructure in place, he (along with any other interested developers) will be able to track down the sources of unwanted timer ticks and make them go away. 

The road to a true solution to the timer-tick problem could be a long one; it is not easy to remove an assumption that has been fundamental to the kernel's design from the beginning. Once the work is done, though, the result should be more than just a fully isolated mode for a small niche use case; it should result in better performance for more ordinary systems as well. The timer tick is, to a great extent, a holdover from the days before Linux existed; there may soon come a time when we don't need it much anymore.
