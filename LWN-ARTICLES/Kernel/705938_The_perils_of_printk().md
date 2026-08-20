---
title: The perils of printk()
url: https://lwn.net/Articles/705938/
date: "November 9, 2016"
category: Kernel messages; printk
author: "By Jonathan Corbet November 9, 2016 2016 Kernel Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 9, 2016

* * *

[2016 Kernel Summit](<https://lwn.net/Articles/KernelSummit2016/>)

One might be tempted to think that there is little to be said about the kernel's `printk()` function; all it does, after all, is output a line of text to the console. But `printk()` has its problems. In a Kernel Summit presentation, Sergey Senozhatsky said that he is simply unable to use `printk()` in its current form. The good news, he said, is that it is not unfixable — and that there are plans for addressing its problems. 

#### Locking the system with printk()

One of the biggest problems associated with `printk()` is deadlocks, which can come about in a couple of ways. One of those is reentrant calls. Consider an invocation of `printk()` that is preempted by a non-maskable interrupt (NMI). The handler for that NMI will, likely as not, want to print something out; NMIs are extraordinary events, after all. If the preempted `printk()` call holds a necessary lock, the second call will deadlock when it tries to acquire the same lock. That is just the sort of unpleasantness that operating system developers normally go far out of their way to avoid. 

This particular problem has been solved; `printk()` now has a special per-CPU buffer that is used for calls in NMI context. Output goes into that buffer and is flushed after the NMI completes, avoiding the need to acquire the locks normally needed by a `printk()` call. 

Unfortunately, `printk()` deadlocks do not end there. It turns out that `printk()` calls can be recursive, the usual ban on recursion in the kernel notwithstanding. Recursive calls can happen as the result of warnings issued from deep within the kernel; lock debugging was also listed as a way to create `printk()` calls at inopportune times. If something calls `printk()` at the wrong time, the result is a recursive call that can deadlock in much the same way as preempted calls. 

The problem looks similar to the NMI case, so it should not be surprising that the solution is similar as well. Sergey has [a proposal](<https://lwn.net/Articles/705941/>) to extend the NMI idea, creating more per-CPU buffers for `printk()` output. Whenever `printk()` wanders into a section of code where recursion could [![\[Sergey Senozhatsky\]](https://static.lwn.net/images/conf/2016/ks/SergeySenozhatsky-sm.jpg)](<https://lwn.net/Articles/705953/>) happen, output from any recursive calls goes to those buffers, to be flushed at a safe time. Two new functions, `printk_safe_enter()` and `printk_safe_exit()`, mark the danger areas. Perhaps confusingly, `printk_safe_enter()` does _not_ mark a safe area; instead, it marks an area where the "safe" output code must be used. 

Given that the per-CPU buffers are required in an increasing number of situations, Peter Zijlstra wondered whether `printk()` should just use the per-CPU buffer always. Sergey responded that this approach is under consideration. 

Hannes Reinecke said that part of the problem results from the two distinct use cases for `printk()`: "chit-chat" and "the system is about to die now." The former type of output can go out whenever, while the latter is urgent. In the absence of better information, `printk()` must assume that everything is urgent, but a lot of problems could be solved by simply deferring non-urgent output to a safe time. Linus Torvalds pointed out that the log level argument should indicate which output is urgent, but Peter said that just deferring non-urgent output is not close to a full solution. The real problem, he said, is in the console drivers; this subject was revisited later in the session. 

One problem with deferring non-urgent output, Sergey said, is that the ordering of messages can be changed and it can be hard to sort them out again. Peter suggested that this was not much of a problem; Hannes said, rather forcefully, that `printk()` output has timestamps on it, so placing it back into the proper order should not be difficult. The problem there, according to Linus, is that timestamps are not necessarily consistent across CPUs; if a thread migrates, the ordering of its messages could be wrong. 

Petr Mladek, who joined Sergey at the front of the room, said that there is a problem with per-CPU buffers: they will almost necessarily be smaller than a single, global buffer, and can thus limit the amount of output that can be accumulated. So it is more likely that the system will lose messages if it is using per-CPU buffers. It was pointed out that the ftrace subsystem has solved this problem for a long time, but it was also pointed out that the cost of that solution is a lot of complicated ring-buffer code. Linus said that the one thing that must be carefully handled is oops messages resulting from a kernel crash; those must make it immediately to the console. 

Sergey went on to say that there is a larger set of `printk()` deadlocks that needs to be dealt with. Thus far, the conversation had concerned "internal" locks that are part of `printk()` itself. But `printk()` often has to acquire "external" locks in other parts of the kernel. The biggest problem area appears to be sending output to the console; there are locks and related problems in various serial drivers that can, once again, deadlock the system. Unlike internal locks, external locks are not controlled by `printk()`, so the problem is harder to solve. 

The kernel already has a `printk_deferred()` function that goes out of its way to avoid taking external locks, once again deferring output to a safer time. Sergey's proposal is to make `printk()` always behave like `printk_deferred()`, eliminating the distinction between the two and enabling the eventual removal of `printk_deferred()` itself. The only exception would be for emergency output, which will always go directly to the console. Linus suggested going one step further, and taking the deferred path even in emergencies, but then flushing the buffers immediately thereafter. 

#### Console troubles and more

Locks are not the only problem with `printk()`, though. To output its messages, it must call into the console drivers and, at completion, it must call [`console_unlock()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/kernel/printk/printk.c#n2335>) which will, among other things, flush any pending output to the console. This function has some unfortunate properties: it can loop indefinitely, it may not be preemptible, and the time it takes depends on the speed of the console — which may not be fast at all. As a result, nobody knows how long a `printk()` call will take, so it's not really safe to call it in any number of situations, including atomic context, RCU critical sections, interrupt context, and more. 

To get around this kind of problem, Jan Kara has [proposed](<https://lkml.org/lkml/2016/4/4/389>) making `printk()` completely asynchronous. Once again, output would be directed to a buffer and sent to the console later, but, with this proposal, the actual writing to the console would be done in a dedicated kernel thread. A call to `printk()` would simply store the message, then use the [`irq_work`](<https://lwn.net/Articles/411605/>) mechanism to kick off that thread. This suggestion passed by without much in the way of complaints from the group in the room. 

Then, there is the problem of `pr_cont()`, a form of `printk()` used to print a single line using multiple calls. This function is not safe on SMP systems, with the result that output generated with it can be mixed up and corrupted. There is a strong desire to get rid of the "continuation line" style of printing, but, as Sergey pointed out, the number of `pr_cont()` calls in the kernel is growing rapidly. The problem, as Linus pointed out, is that there is no other convenient way to output variable-length lines in the kernel. Changing `pr_cont()`, to use a per-CPU buffer, for example, is possible, but one would want to create a well thought-out helper function. Then, perhaps, `pr_cont()` users could be easily fixed up with a Coccinelle script. 

Ted Ts'o asked how much of a problem interleaved output really is on a production system; the consensus seemed to be that it was rarely a problem. Linus said that, on occasion, he sees ugly oops output as a result of continuation lines. Andy Lutomirski said, with a grin, that his algorithm for dealing with interleaved oops output is to wait for Linus to straighten it out for him. That solution seemed to work for the group as a whole; there does not seem to be any work planned in this area in the immediate future. 

The final topic, covered in a bit of a hurry at the end of the session, was the `console_sem` semaphore. This semaphore covers access to all consoles in the system, so it is a global contention point. But there are paths that acquire `console_sem` that do not need to modify the console list or even write to a console. For example, simply reading `/proc/consoles` from user space will acquire that semaphore. That can cause unpleasant delays, including in `printk()` itself. And releasing this semaphore, once again, results in a call to `console_unlock()`, with the same associated problems. 

Sergey suggested that `console_sem` should be turned into a reader/writer lock. That way, any path that does not need to modify the console list itself can acquire the lock in reader mode, increasing parallelism. That still won't help direct callers of `console_unlock()`, who will still be stuck flushing output to the device. For them, there was discussion of splitting `console_unlock()` into synchronous and asynchronous versions; the latter would just wake the `printk()` thread rather than flushing any pending console output itself. There does not appear to be any urgency to this work, though. 

That is where time ran out and the session ended. [Sergey's slides](<http://www.slideshare.net/SergeySENOZHATSKY/printk-considered-harmful>) are available for those who are interested.
