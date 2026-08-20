---
title: Simple wait queues
url: https://lwn.net/Articles/577370/
date: "December 18, 2013"
category: Wait queues
author: "By Jonathan Corbet December 18, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 18, 2013

A "wait queue" in the Linux kernel is a data structure to manage threads that are waiting for some condition to become true; they are the normal means by which threads block (or "sleep") in kernel space. Over the years, the wait queue mechanism has evolved into a fairly elaborate and complicated kernel subsystem. Now, however, there is a move afoot to simplify that code, using a wait queue variant developed for the realtime tree; the result could be a fair amount of code churn in the kernel. 

A look at [`<linux/wait.h>` from the 2.0 kernel](<https://lwn.net/Articles/577374/>) reveals a simple data structure: a basic linked list of waiting threads. A `wake_up()` call on a wait queue would walk the list, putting each thread into the runnable state; there was not a whole more to it than that. Then, in 1999, the infamous [Mindcraft study](<http://www.mindcraft.com/whitepapers/first-nts4rhlinux.html>) pointed out some performance deficiencies in Linux; one of those was the "thundering herd" problem where multiple processes would be awakened and contend for a resource that only one of them could obtain. As a result, the "exclusive wait" functionality — where only the first of possibly many waiting threads would wake — was added. Then a callback mechanism was added in the 2.5 series so that the new asynchronous I/O facility could step in when things would otherwise block. And so on. 

The end result is a data structure that is far larger and more complex than it was in the 2.0 days. It is the callback feature that was most problematic for the realtime tree, though; since those callbacks can sleep, they prevent the use of "raw" spinlocks to protect the wait queues themselves. To work around this problem, Thomas Gleixner created a new "simple wait queue" mechanism that would dispense with most of the added functionality and, thus, be suitable for use in the realtime kernel. 

The [2013 Realtime Linux Workshop](<https://lwn.net/Articles/572740/>) identified this code as a candidate for a relatively easy move into the mainline. In response, Paul Gortmaker has extracted the simple wait queue facility and [posted the resulting patch series](<https://lwn.net/Articles/576773/>) for review. 

The code looks a lot like a return to the 2.0 kernel; much of the functionality that wait queues have gained in the meantime has been stripped away, leaving a familiar-looking linked list of waiting threads. There is no exclusive wakeup feature, no callback feature, and not much of anything else. What there _is_ , though, is a wait queue mechanism that is sufficient for the needs of most wait queue users (of which there are many) in the kernel. 

The API is similar to that of existing wait queues. Wait queue entries and wait queue heads are defined with: 

```
#include <linux/swait.h>
    
        DEFINE_SWAITER(name);
        DEFINE_SWAIT_HEAD(name);
```

The low-level API, which requires a direct call to `schedule()` to put the calling thread to sleep, looks like this: 

```
void swait_prepare(struct swait_queue_head *head, struct swaiter *w, int state);
        void swait_finish(struct swait_queue_head *head, struct swaiter *w);
```

The `swait_prepare()` call is used to add the process to the given wait queue `head` and put it into the appropriate sleeping `state`. After performing any necessary checks and calling `schedule()`, the newly woken thread will call `swait_finish()` to remove itself from the queue and clean up. 

The current wait queue implementation has an extensive set of macros to simplify the task of waiting for a condition; there is a similar, but much smaller set for simple wait queues: 

```
void swait_event(queue, condition);
        int swait_event_interruptible(queue, condition);
        void swait_event_timeout(queue, condition, timeout);
        int swait_event_interruptible_timeout(queue, condition, timeout);
```

Most of the other versions of `wait_event()`, including the ["killable" variants](<https://lwn.net/Articles/288056/>), are not provided. It is amusing to look at a list of `wait_event()` macros that lack equivalents in the new API, just to see how this interface has grown over the years: 

```
wait_event_cmd(wq, condition, cmd1, cmd2);
        wait_event_hrtimeout(wq, condition, timeout);
        wait_event_killable(wq, condition);
        wait_event_lock_irq_cmd(wq, condition, lock, cmd);
        wait_event_lock_irq(wq, condition, lock);
        wait_event_interruptible_hrtimeout(wq, condition, timeout);
        wait_event_interruptible_exclusive(wq, condition);
        wait_event_interruptible_locked(wq, condition);
        wait_event_interruptible_locked_irq(wq, condition);
        wait_event_interruptible_exclusive_locked(wq, condition);
        wait_event_interruptible_exclusive_locked_irq(wq, condition);
        wait_event_interruptible_lock_irq_cmd(wq, condition, lock, cmd);
        wait_event_interruptible_lock_irq(wq, condition, lock);
        wait_event_interruptible_lock_irq_timeout(wq, condition, lock, timeout);
```

There is little impediment to adding "simple" versions of most of the above macros should the need arise; it will be interesting to see how many of them show up in the coming years. Needless to say, there is also nothing like [the archaic `sleep_on()` interface](<https://lwn.net/Articles/68118/>); it is safe to say nobody will try to add a version of that. 

Paul's posting notes that adding the simple wait queues makes the kernel smaller, even when they are only used in a couple of places. Given the size reduction and the relative simplicity of the interface, it is unsurprising that there has been no opposition to adding this code so far. The only real question is how that addition is to be done. Christoph Hellwig [suggested](<https://lwn.net/Articles/577386/>) that the simple wait queues could simply replace the current implementation, with the few places needing the fancier functionality being changed to use the older code under a new name. Paul, though, [worried](<https://lwn.net/Articles/577387/>) that such a wholesale change would create a flag day with problems being associated with the wait queue change in mysterious ways. 

Nobody wants that kind of situation, so it seems more likely that simple wait queues will retain their "swait" naming scheme. The kernel _might_ see a wholesale naming change for the existing wait queues to make it clear that there is now a choice to be made, though. Thus, we may see a large patch changing `wait_event()` to `cwait_event()`, and so on, without changing functionality; after that, individual call sites could be changed to simple wait queues at leisure. The result would be a fair amount of code churn, but that churn should leave a smaller and simpler kernel in its wake.
