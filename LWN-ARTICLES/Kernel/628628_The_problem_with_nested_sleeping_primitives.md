---
title: The problem with nested sleeping primitives
url: https://lwn.net/Articles/628628/
date: "January 7, 2015"
category: Race conditions
author: "By Jonathan Corbet January 7, 2015"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 7, 2015

Waiting for events in an operating system is an activity that is fraught with hazards; without a great deal of care, it is easy to miss the event that is being waited for. The result can be an infinite wait — an outcome which tends to be unpopular with users. The kernel has long since buried the relevant code in the core kernel with the idea that, with the right API, wait-related race conditions can be avoided. Recent experience shows, though, that the situation is not always quite that simple. 

Many years ago, kernel code that needed to wait for an event would execute something like this: 

```
while (!condition)
     	sleep_on(&wait_queue);
```

The problem with this code is that, should the condition become true between the test in the `while` loop and the call to `sleep_on()`, the wakeup could be lost and the sleep would last forever. For this reason, `sleep_on()` was deprecated for a long time and no longer exists in the kernel. 

The contemporary pattern looks more like this: 

```
DEFINE_WAIT(wait);
    
        while (1) {
        	prepare_to_wait(&queue, &wait, state);
        	if (condition)
    	    break;
    	schedule();
        }
        finish_wait(&queue, &wait);
```

Here, `prepare_to_wait()` will enqueue the thread on the given `queue` and put it into the given execution `state`, which is usually either `TASK_INTERRUPTIBLE` or `TASK_UNINTERRUPTIBLE`. Normally, that state will cause the thread to block once it calls `schedule()`. If the wakeup happens first, though, the process state will be set back to `TASK_RUNNING` and `schedule()` will return immediately (or, at least, as soon as it decides this thread should run again). So, regardless of the timing of events, this code should work properly. The numerous variants of the `wait_event()` macro expand into a similar sequence of calls. 

Signs of trouble can be found in messages like the following, which are turning up on systems running the 3.19-rc kernels: 

```
do not call blocking ops when !TASK_RUNNING; state=1
         	set at [<ffffffff910a0f7a>] prepare_to_wait+0x2a/0x90
```

This message, the result of some new checks added for 3.19, is indicating that a thread is performing an action that could block while it is ostensibly already in a sleeping state. One might wonder how that can be, but it is not that hard to understand in the light of the sleeping code above. 

The "`condition`" checked in that code is often a function call; that function may perform a fair amount of processing on its own. It may need to acquire locks to properly check for the wakeup condition. That, of course, is where the trouble comes in. Should the condition-checking function call something like `mutex_lock()`, it will go into a new version of the going-to-sleep code, changing the task state. That, of course, may well interfere with the outer sleeping code. For this reason, nesting of sleeping primitives in this way is discouraged; the new warning was added to point the finger at code performing this kind of nesting. It turns out that kind of nesting happens rather more often than the scheduler developers would have liked. 

So what is a developer to do if the need arises to take locks while checking the sleep condition? [One solution](<http://git.kernel.org/linus/61ada528dea028331e99e8ceaed87c683ad25de2>) was added in 3.19; it takes the form of a new pattern that looks like this: 

```
DEFINE_WAIT_FUNC(wait, woken_wait_function);
    
        add_wait_queue(&queue, &wait);
        while (1) {
    	if (condition)
    	    break;
    	wait_woken(&wait, state, timeout);
        }
        remove_wait_queue(&queue, &wait);
```

The new `wait_woken()` function encapsulates most of the logic needed to wait for a wakeup. At a first glance, though, it looks like it would suffer from the same problem as `sleep_on()`: what happens if the wakeup comes between the condition test and the `wait_woken()` call? The key here is in the use of a special wakeup function called `woken_wait_function()`. The `DEFINE_WAIT_FUNC()` macro at the top of the above code sequence associates this function with the wait queue entry, changing what happens when the wakeup arrives. 

In particular, that change causes a special flag (`WQ_FLAG_WOKEN`) to be set in the `flags` field of the wait queue entry. If `wait_woken()` sees that flag, it knows that the wakeup already occurred and doesn't block. Otherwise, the wakeup has not occurred, so `wait_woken()` can safely call `schedule()` to wait. 

This pattern solves the problem, but there is a catch: every place in the kernel that might be using nested sleeping primitives needs to be found and changed. There are a _lot_ of places to look for problems and potentially fix, and the fix is not an easy, mechanical change. It would be nicer to come up with a version of `wait_event()` that doesn't suffer from this problem in the first place or, failing that, with something new that can be easily substituted for `wait_event()` calls. 

Kent Overstreet [thinks he has that replacement](<https://lwn.net/Articles/628632/>) in the form of the "closure" primitive used in the bcache subsystem. Closures work in a manner similar to `wait_woken()` in that the wakeup state is stored internally to the relevant data structure; in this case, though, an atomic reference count is used. Interested readers can see [`drivers/md/bcache/closure.h`](<https://lwn.net/Articles/628633/>) and [`closure.c`](<https://lwn.net/Articles/628634/>) for the details. Scheduler developer Peter Zijlstra [is not convinced](<https://lwn.net/Articles/628635/>) about the closure code, but he agrees that it would be nice to have a better solution. 

The form of that solution is thus unclear at this point. What does seem clear is that the current nesting of sleeping primitives needs to be fixed. So, one way or another, we are likely to see a fair amount of work going into finding and changing problematic calls over the next few development cycles. Until that work is finished, warnings from the new debugging code are likely to be a common event.
