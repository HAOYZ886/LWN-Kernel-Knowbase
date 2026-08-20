---
title: Deleting timers quickly
url: https://lwn.net/Articles/84836/
date: "May 12, 2004"
category: Timers; del timer
author: 
---

Kernel timers are a mechanism which allows kernel code to request that a function be called, in software interrupt context, after a given period of time has passed. They are heavily used for all sorts of delays and deferred actions within the kernel. The timer interface has been relatively stable for some time; it has not changed greatly in 2.6. [Linux Device Drivers, Chapter 6](<http://www.xml.com/ldd/chapter/book/ch06.html#t5>) covers the timer interface in some detail. 

Often, kernel code which has queued a timer finds that it needs to delete that timer. There are two functions which perform this task: 

```
int del_timer(struct timer_list *timer);
        int del_timer_sync(struct timer_list *timer);
```

`del_timer()` ensures that the given timer is not queued to run anywhere in the system; it returns a non-zero value if the timer actually had to be dequeued. `del_timer_sync()` performs the same function, but it also guarantees that the timer is not actually running on any processor in the system; it will block the current process if necessary while it waits for a running timer to complete. The stronger guarantee is often needed; an unexpected timer running in the corner can create no end of unpleasant race conditions. 

Geoff Gustafson recently [discovered](<https://lwn.net/Articles/84839/>) that `del_timer_sync()` was one of the biggest kernel CPU hogs on a 32-processor NUMA system running "an enterprise database application." The problem is that `del_timer_sync()` must query each processor to ensure that the given timer is not currently running there. As the number of processors grows, this query loop takes longer to run. The situation is even worse on NUMA systems, since the loop must look at non-local (read "slow") memory for each processor. 

Geoff posted a patch which solved the problem by remembering where each timer last ran. Since the kernel does not move timers across processors, the query loop in `del_timer_sync()` could then be reduced to looking at the single processor where the timer would have to be. It was observed, however, that a simpler solution is possible: 

```
if (! del_timer(timer))
            /* Do the full CPU query loop */
```

The idea here is that, if the timer was successfully deleted from the queue before it ran, there is no need to check to see if it is running anywhere. The only problem with this idea is that it is wrong. Timer functions can - and often do - resubmit themselves. If the timer to be deleted has resubmitted itself, but is still running, the above code will fail. If kernel code is deleting a timer, it really should first ensure that said timer will not resubmit itself, but the timer code cannot count on that behavior. 

That said, some of the top callers of `del_timer_sync()` within the kernel are using timers which do not resubmit themselves. There is no reason why that code should pay the overhead of a full system search when, if a timer has been deleted off the queue before running, it is already guaranteed that the timer will not be running on any processor. For cases like this, a new function has been created: 

```
int del_singleshot_timer_sync(struct timer_list *timer);
```

Callers of this function must guarantee that the timer does not resubmit itself; in its current form, `del_singleshot_timer_sync()` will generate an oops if it detects a resubmitted timer. This function has not yet found its way into the mainline, but, given that it can yield a performance improvement of 2-3 orders of magnitude on large NUMA systems, its addition seems likely.
