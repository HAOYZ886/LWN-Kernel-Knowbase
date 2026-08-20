---
title: TASK_KILLABLE
url: https://lwn.net/Articles/288056/
date: "July 1, 2008"
category: Scheduler; TASK KILLABLE
author: "By Jonathan Corbet July 1, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 1, 2008

Like most versions of Unix, Linux has two fundamental ways in which a process can be put to sleep. A process which is placed in the `TASK_INTERRUPTIBLE` state will sleep until either (1) something explicitly wakes it up, or (2) a non-masked signal is received. The `TASK_UNINTERRUPTIBLE` state, instead, ignores signals; processes in that state will require an explicit wakeup before they can run again. 

There are advantages and disadvantages to each type of sleep. Interruptible sleeps enable faster response to signals, but they make the programming harder. Kernel code which uses interruptible sleeps must always check to see whether it woke up as a result of a signal, and, if so, clean up whatever it was doing and return `-EINTR` back to user space. The user-space side, too, must realize that a system call was interrupted and respond accordingly; not all user-space programmers are known for their diligence in this regard. Making a sleep uninterruptible eliminates these problems, but at the cost of being, well, uninterruptible. If the expected wakeup event does not materialize, the process will wait forever and there is usually nothing that anybody can do about it short of rebooting the system. This is the source of the dreaded, unkillable process which is shown to be in the "D" state by `ps`. 

Given the highly obnoxious nature of unkillable processes, one would think that interruptible sleeps should be used whenever possible. The problem with that idea is that, in many cases, the introduction of interruptible sleeps is likely to lead to application bugs. As recently [noted](<https://lwn.net/Articles/288062/>) by Alan Cox: 

Unix tradition (and thus almost all applications) believe file store writes to be non signal interruptible. It would not be safe or practical to change that guarantee. 

So it would seem that we are stuck with the occasional blocked-and-immortal process forever. 

Or maybe not. A while back, Matthew Wilcox realized that many of these concerns about application bugs do not really apply if the application is about to be killed anyway. It does not matter if the developer thought about the possibility of an interrupted system call if said system call is doomed to never return to user space. So Matthew created a new sleeping state, called `TASK_KILLABLE`; it behaves like `TASK_UNINTERRUPTIBLE` with the exception that fatal signals will interrupt the sleep. 

With `TASK_KILLABLE` comes a new set of primitives for waiting for events and acquiring locks: 

```
int wait_event_killable(wait_queue_t queue, condition);
    	long schedule_timeout_killable(signed long timeout);
    	int mutex_lock_killable(struct mutex *lock);
    	int wait_for_completion_killable(struct completion *comp);
    	int down_killable(struct semaphore *sem);
```

For each of these functions, the return value will be zero for a normal, successful return, or a negative error code in case of a fatal signal. In the latter case, kernel code should clean up and return, enabling the process to be killed. 

The `TASK_KILLABLE` patch was merged for the 2.6.25 kernel, but that does not mean that the unkillable process problem has gone away. The number of places in the kernel (as of 2.6.26-rc8) which are actually using this new state is quite small - as in, one need not worry about running out of fingers while counting them. The NFS client code has been converted, which can only be a welcome development. But there are very few other uses of `TASK_KILLABLE`, and none at all in device drivers, which is often where processes get wedged. 

It can take time for a new API to enter widespread use in the kernel, especially when it supplements an existing functionality which works well enough most of the time. Additionally, the benefits of a mass conversion of existing code to killable sleeps are not entirely clear. But there are almost certainly places in the kernel which could be improved by this change, if users and developers could identify the spots where processes get hung. It also makes sense to use killable sleeps in new code unless there is some pressing reason to disallow interruptions altogether.
