---
title: Improving ptrace()
url: https://lwn.net/Articles/432114/
date: "March 8, 2011"
category: ptrace
author: "By Jonathan Corbet March 8, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 8, 2011

The `ptrace()` system call is rarely held up as one of the better parts of the Unix interface. This call, which is used for the tracing and debugging of processes, is gifted with strange semantics (it reparents the traced process to the tracer), numerous interface warts, and occasionally unpredictable behavior. It is also hard to implement within the kernel; there are few developers who are willing to get into the depths of the `ptrace()` implementation. So it's not surprising that there has been occasional talk of simply replacing `ptrace()` with something better; see [this 2010 article](<https://lwn.net/Articles/371501/>) for a description of one such discussion. 

While some developers think that `ptrace()` is beyond repair, Tejun Heo disagrees. To back that up, he has posted [a set of proposals](<https://lwn.net/Articles/430373/>) for the improvement of the interface, saying: 

ptrace currently is in a pretty bad shape and I think one of the biggest reasons is a lot of effort has been spent trying to come up with something completely new instead of concentrating on improving what's already there. I think the existing principles are pretty sound. They just need some love and attention here and there. 

The bulk of the "love and attention" Tejun means to apply is addressed at the interaction between tracing and job control. In an untraced process, job control is used by the kernel and the shell to stop and restart processes, possibly moving them between the foreground and the background. Adding tracing to the picture confuses things for a number of reasons. For example, reparenting the traced process deprives the real parent of the ability to get notifications when that process is stopped or started. There are also some strange internal transitions between the `TASK_STOPPED` and `TASK_TRACED` states which lead to unpredictable and sometimes surprising behavior. For example, a task which is running under `strace` can be stopped with `^Z` as usual, but the shell will be unable to restart it. 

Tejun has a series of concrete proposals to improve the situation. The first of these is that a traced process should always, when stopped, be in the `TASK_TRACED` state. The current strange transitions between that state and `TASK_STOPPED` would go away. He would fix things so that notifications when a process stops or starts would always go to the real parent, even when a process has been reparented for tracing. Some edge cases, such as what happens when a traced process is detached, would be fixed so that process's behavior matches the untraced case. 

To fix the "can't start a stopped, traced process" problem, Tejun would further enshrine the rule that the tracing process has total control over the traced process's state. So it's up to the tracer to start a stopped process if the shell wants that done. Currently, tracers have no way to know that the real parent has tried to start a stopped process, so a notification mechanism needs to be added. That would be done by extending the `STOPPED` notification that can currently be obtained with one of the variants of the `wait()` system call. 

Finally, Tejun would like to fix the behavior of the `PTRACE_ATTACH` operation, which attaches to a process and sends a `SIGSTOP` signal to put it into the stopped state. The signal confuses things, and the stopped state is undesirable; it is not really possible, though, to change the semantics of `PTRACE_ATTACH` in this way without breaking applications. So he would create a new `PTRACE_SEIZE` operation which would attach to a process (if it's not already attached) and put the process immediately into the `TASK_TRACED` state. 

These changes, Tejun thinks, are enough to turn `ptrace()` into something rather more predictable and civilized. He'd like to go forward into the implementation with a 2.6.40 target for merging. In the following discussion, it seems that most developers agree with these changes, modulo a quibble or two. The [one big exception](<https://lwn.net/Articles/432132/>) was Roland McGrath, who has done a lot of work in this area. Roland has some different ideas, especially with regard to `PTRACE_SEIZE`. 

Roland's alternative to `PTRACE_SEIZE` (if it can truly be called an "alternative," having been suggested first) is to add two new commands: `PTRACE_ATTACH_NOSTOP` and `PTRACE_INTERRUPT`. The former would attach to a process but not change its running state in any way, while the latter would stop the process and put it into the `TASK_TRACED` state. He sees a number of advantages to this approach, including the ability to trace a process without ever stopping it. There are cases (`strace` comes to mind) where there is no need to stop the process; avoiding doing so allows the process to be traced while minimizing the effects on its behavior. 

Roland also foresaw a variant of `PTRACE_INTERRUPT` which would only stop a process when it's running in user space. That would avoid the occasional "interrupted system call" failure that current tracing can cause. He also worries about what happens when `PTRACE_SEIZE` is, itself, interrupted; handling that situation in a way that supports the writing of robust applications, he says, would be hard. Finally, he raises the issue of scalability; he does not think that `PTRACE_SEIZE` will work well for the debugging of highly threaded applications. In summary, he said: 

None of this means at all that PTRACE_SEIZE is worthless. But it is certainly inadequate to meet the essential needs that motivate adding new interfaces in this area. The PTRACE_ATTACH_NOSTOP idea I suggested is far from complete for all the issues as well, but it is a more versatile building block than PTRACE_SEIZE. 

Unfortunately, it seems that Roland is changing jobs and stopping work in this area, so his thoughts may carry less weight than they normally would have. As of this writing, there have been few responses to his post; Tejun has mostly [dismissed](<https://lwn.net/Articles/432412/>) Roland's concerns. Tejun has also posted a patch series implementing parts of his proposal, but not, yet, `PTRACE_SEIZE`. The uncontroversial parts of this work will almost certainly be merged; how `PTRACE_ATTACH` will be fixed in the end remains to be seen.
