---
title: "Seccomp user-space notification and signals"
url: https://lwn.net/Articles/851813/
date: "April 9, 2021"
category: "Security-seccomp"
author: "By Jonathan Corbet April 9, 2021"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 9, 2021

The [`seccomp()` mechanism](<https://man7.org/linux/man-pages/man2/seccomp.2.html>) allows the imposition of a filter program (expressed in ["classic" BPF](<https://lwn.net/Articles/599755/>)) that makes policy decisions on whether to allow each system call invoked by the target process. The [user-space notification feature](<https://lwn.net/Articles/756233/>) further allows those decisions to be deferred to another process. As [this recent patch set](<https://lwn.net/ml/linux-kernel/20210318051733.2544-1-sargun@sargun.me/>) from Sargun Dhillon shows, though, user-space notification still has some rough edges, especially when it comes to signals. This patch makes a simple change to try to address a rather complex problem brought to the fore by changes in the Go language's preemption model. 

Normally, `seccomp()` is used to implement a simple sort of attack-surface reduction, making much of the system-call space off limits for the affected process. User-space notification can be used to that end, but the objective there is often different: it allows a supervisor process to emulate system calls for the target process. An example might be a container manager that wishes to make [`mount()`](<https://man7.org/linux/man-pages/man2/mount.2.html>) available inside a container, but with some strict limits on what can actually be mounted. User-space notification allows the (privileged) supervisor to actually perform the mount operations it approves of and return the results to the target process. 

While the supervisor is handling an intercepted system call, the target process will be blocked in the kernel, waiting for a response to come back. Should that process receive a signal, though, it will stop waiting and respond immediately to the signal; if the signal itself is not fatal, the result may well be the system call returning an `EINTR` error to the target process. The supervisor, instead, will not know about the signal until it tries to give the kernel its answer to the original notification; at that point, it will get an `ENOENT` error indicating that the notification is no longer alive. 

This sort of interruption can be inconvenient, especially if the supervisor has carried out some sort of long task on the target's behalf. If the signal does not kill the target process, it is likely that the same operation will be retried shortly, leading to extra work being done. Most of the time, though, non-fatal signals of this type are likely to be rare in programs running under `seccomp()` monitoring. 

#### Go signal

More accurately, that was once true, but the developers of the Go language had a problem of their own to solve. That language's "goroutine" lightweight thread model requires that the Go runtime handle scheduling, switching between goroutines as needed so that they all get a chance to run. Beyond that, there is a need for occasional "stop the world" events where all goroutines are paused so that the garbage collector can do its job. This has been handled by having the compiler put preemption checks at the beginning of each function. 

What happens, though, if a goroutine runs for a long time without calling any functions? This can happen if the routine is running inside of some sort of tight loop; in the worst case, that loop could be spinning on a lock and preventing the lock holder from running to release it, a situation that tends to increase the overall level of user disgruntlement. Another way to delay preemption is to make a long-running system call. 

The Go developers have tried a few ways of solving this problem. One of them involved inserting preemption checks at backward jumps in the code (thus at the end of a loop, for example). Even when that check was reduced to a single instruction, the resulting performance penalty was deemed to be too high; this approach also doesn't help in the long-running system-call case. So the Go community decided to address this problem with a [non-cooperative preemption mechanism](<https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md>) instead. In simple terms, any goroutine that runs for 10ms without yielding will receive a `SIGURG` signal from the runtime, which will then reschedule the thread, initiate garbage collection, or do whatever else needs to be done at that time. 

System calls that end up being referred to another process via `seccomp()` tend to run longer than usual, and the sorts of tasks that a supervisor process might carry out — mounting a filesystem, for example — can take longer yet. This has evidently led to a lot of interrupted, `seccomp()`-mediated system calls in Go programs and an associated desire to find a way to stop those interruptions. 

#### Masking non-fatal signals

To address this problem, Dhillon's patch set adds a new flag (called `SECCOMP_USER_NOTIF_FLAG_WAIT_KILLABLE`) to the `SECCOMP_IOCTL_NOTIF_RECV` `ioctl()` command that is used by the supervisor process to receive notifications. If that flag is set when a notification is given to the supervisor, the target process will be put into a "killable" wait, meaning that fatal signals will still be delivered, but any others will be masked until after the supervisor has responded to the notification. Non-fatal signals will thus no longer interrupt system calls while the supervisor process is working on them. 

Note that if a non-fatal signal arrives _before_ the supervisor reads the notification, the target's system call will be interrupted as usual. The notification will be canceled, and the supervisor will get an error if it tries to read that notification. The end result in that case is as if the system call never happened in the first place. Once the notification is delivered, though, the system call will run to completion. It is a relatively small change that solves this problem, though that solution comes at the expense of adding arbitrary delays to Go's preemption mechanism when `seccomp()` and user-space notification are in use. That is just the sort of delay that the preemption mechanism was created to prevent, but it will at least be under the control of the supervisor and, presumably, bounded. 

This patch set has been posted twice as of this writing; it has not received much in the way of responses. That may suggest that few people have looked at it so far, which is not an ideal situation for a security-related change to the user-space API. Until that situation changes, this work seems unlikely to advance and users of Go with `seccomp()` and user-space notifications will continue to have problems.
