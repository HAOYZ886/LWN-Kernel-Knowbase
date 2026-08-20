---
title: Improving printk()
url: https://lwn.net/Articles/737822/
date: "November 1, 2017"
category: Kernel messages; printk
author: "By Jonathan Corbet November 1, 2017 2017 Kernel Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 1, 2017

* * *

[2017 Kernel Summit](<https://lwn.net/Articles/KernelSummit2017/>)

When a kernel developer wants to communicate a message to user space, be it for debugging or to report a serious problem with the system, the venerable `printk()` function is usually the tool of choice. But, as Steve Rostedt (accompanied by Petr Mladek and Sergey Senozhatsky) noted during a brief session at the 2017 Kernel Summit, `printk()` has not aged well. In particular, it can affect the performance of the system as a whole; the roots of that problem and a possible solution were discussed, but a real solution will have to wait for the appearance of the code. 

The problem, Rostedt said, has to do with the management of the console lock, which serializes access to the console device where messages are printed. [Deep within `printk()`](<http://elixir.free-electrons.com/linux/v4.13.10/source/kernel/printk/printk.c#L1761>), one will find interesting call sequences like: 

```
if (console_trylock())
    	console_unlock();
```

The first call will attempt to acquire the console lock but may not succeed; the second, on its surface, releases that lock immediately after it was acquired. It is the work involved in releasing the console lock that can create problems for the system. 

`printk()` must proceed regardless of the availability of the console lock; since `printk()` is called from all over the kernel, waiting for any sort of lock risks deadlocking the system. So, if a particular call is unable to obtain the console lock, it simply buffers its [![\[Steve Rostedt\]](https://static.lwn.net/images/conf/2017/osse/printk-sm.jpg)](<https://lwn.net/Articles/737823/>) output and returns, in the expectation that somebody else will flush that output to the console. That task falls to the thread that holds the console lock; that thread is expected to flush out all buffered output as part of the process of releasing the lock. 

On a large system with a lot of CPUs, there can be multiple threads calling `printk()` at any given time. They can leave behind a lot of work for the unlucky thread that holds the console lock; indeed, in the worst case, output can continue to pile up while the buffer is being flushed, leaving the lock holder with a job of indefinite duration. That is bad for system performance and the latency of anything that needs to run on the affected CPU. 

Peter Zijlstra jumped in to say that, whenever this problem comes up, he just removes `printk()` calls until it goes away. Andrew Morton, instead, asked for forgiveness for creating this mechanism in the first place; it was, he said, something he came up with at 3AM. Rostedt went on to say that, in the worst case, flushing `printk()` output can take so long that the watchdog fires and the system crashes. If there are 100 CPUs in the system, one of them can end up flushing `printk()` output forever. 

There are, he said, a couple of possible solutions to the problem. One of them is to remove `printk()` calls as Zijlstra suggested, but that is a game of whack-a-mole that is never really done. The alternative is a new locking scheme where the second thread attempting to obtain the console lock spins and waits for it to become available. The current holder of the lock will see that there is a waiter and release the lock; the second thread will then acquire it and continue flushing the output buffer. If multiple CPUs are generating output, the lock will circulate between them, and none will end up printing output for too long. 

Jan Kara said that he had once tried to implement a similar scheme, but he ran into a lot of special cases and finally gave up on it. Mathieu Desnoyers suggested deferring any excess printing work to a workqueue rather than pushing it out immediately; Ben Herrenschmidt concurred, saying that there is no real need to flush output right away. But Rostedt answered that Linus Torvalds insists that crash dumps must go out immediately, so any scheme that can delay output will not fly. The entire `printk()` buffer must be printed out as soon as possible. 

There was some unstructured discussion on the details of the new approach, but no real conclusions were reached. This is a conversation that will have to resume once the code to implement the new mechanism has been posted. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, for supporting his travel to the Kernel Summit.]
