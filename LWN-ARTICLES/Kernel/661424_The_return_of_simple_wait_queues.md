---
title: The return of simple wait queues
url: https://lwn.net/Articles/661424/
date: "October 21, 2015"
category: Wait queues
author: "By Jonathan Corbet October 21, 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 21, 2015

A "wait queue" in the kernel is a data structure that allows one or more processes to wait (sleep) until something of interest happens. They are used throughout the kernel to wait for available memory, I/O completion, message arrival, and many other things. In the early days of Linux, a wait queue was a simple list of waiting processes, but various scalability problems (including the thundering herd problem highlighted by the infamous [Mindcraft report](<http://www.mindcraft.com/whitepapers/nts4rhlinux.html>) in 1999) have led to the addition of a fair amount of complexity since then. The [simple wait queue](<https://lwn.net/Articles/661415/>) patch set is an attempt to push the pendulum back in the other direction. 

Simple wait queues are not new; we [looked at them](<https://lwn.net/Articles/577370/>) in 2013. The API has not really changed since then, so that discussion will not be repeated here. For those who don't want to go back to the previous article, the executive summary is that simple wait queues provide an interface quite similar to that of regular wait queues, but with a lot of the bells and whistles removed. Said bells (or perhaps they are whistles) include exclusive wakeups (an important feature motivated by the aforementioned Mindcraft report), ["killable" waits](<https://lwn.net/Articles/288056/>), high-resolution timeouts, and more. 

There is value in simplicity, of course, and the memory saved by switching to a simple wait queue is welcome, even if it's small. But that, alone, would not be justification for the addition of another wait-queue mechanism to the kernel. Adding another low-level scheduling primitive like this increases the complexity of the kernel as a whole and makes ongoing maintenance of the scheduler harder. It is unlikely to happen without a strong and convincing argument in its favor. 

In this case, the drive for simple wait queues is (as is the code itself) coming from the realtime project. The realtime developers seek determinism at all times, and, as it turns out, current mainline wait queues get in the way. 

The most problematic aspect of ordinary wait queues appears to be the ability to add custom wakeup callbacks. By default, if one of the various `wake_up()` functions is called to wake processes sleeping on a wait queue, the kernel will call `default_wake_function()`, which simply wakes these waiting processes. But there is a mechanism provided to allow specialized users to change the wake-up behavior of wait queues: 

```
typedef int (*wait_queue_func_t)(wait_queue_t *wait, unsigned mode,
        				     int flags, void *key);
        void init_waitqueue_func_entry(wait_queue_t *q, wait_queue_func_t func);
```

This feature is only used in a handful of places in the kernel, but they are important uses. The I/O multiplexing system calls (`poll()`, `select()`, and `epoll_wait()`) use it to turn specific device events into poll events for waiting processes. The [`userfaultfd()` code](<https://lwn.net/Articles/636226/>) (added for the 4.3 release) has a wake function that only does a wakeup for events in the address range of interest. The `exit()` code similarly uses a custom wake function to only wake processes that have an interest in the exiting process. And so on. It is a feature that cannot be removed. 

The problem with this feature, from the realtime developers' point of view, is that they have no control over how long the custom wake function will take to run. This feature thus makes it harder for them to provide response-time guarantees. Beyond that, these callbacks require that the wait-queue structure be protected by an ordinary spinlock, which is a sleeping lock in the realtime tree. That, too, gets in the way in the realtime world; it prevents, for example, the use of `wake_up()` in hard (as opposed to threaded) interrupt handlers. 

Simple wait queues dispense with custom callbacks and many other wait-queue features, allowing the entire data structure to be reduced to: 

```
struct swait_queue_head {
    	raw_spinlock_t		lock;
    	struct list_head	task_list;
        };
    
        struct swait_queue {
    	struct task_struct	*task;
    	struct list_head	task_list;
        };
```

The `swait_queue_head` structure represents the wait queue as a whole, while `struct swait_queue` represents a process waiting in the queue. Waiting is just a matter of adding a new `swait_queue` entry to the list, and wakeups are a simple traversal of that list. Regular wait queues, instead, may have to search the list for specific processes to wake. The lack of custom wakeup callbacks means that the time required to wake any individual process on the list is known (and short), so a raw spinlock can be used to protect the whole thing. 

This patch set has been posted by Daniel Wagner, who has taken on the challenge of getting it into the mainline, but the core wait-queue work was done by Peter Zijlstra. It has seen a few revisions in the last few months, but comments appear to be slowing down. One never knows with such things (the patches looked mostly ready in 2013 as well), but it seems like there is not much keeping this work from going into the 4.4 kernel.
