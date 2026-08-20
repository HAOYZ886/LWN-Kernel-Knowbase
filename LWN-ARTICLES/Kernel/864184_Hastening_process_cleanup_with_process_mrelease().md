---
title: Hastening process cleanup with process_mrelease()
url: https://lwn.net/Articles/864184/
date: "July 26, 2021"
category: "Memory management-Out-of-memory handling; System calls-process mrelease"
author: "By Jonathan Corbet July 26, 2021"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 26, 2021

One of the fundamental invariants of computing is that, regardless of how much memory is installed in a system, it is never enough. This is especially true of systems with tight performance constraints, where every page of memory is allocated and in use, making it difficult to find more when it is badly needed. One way to make more memory available is to kill one or more processes, freeing their resources for other users. But that often does not work as quickly or reliably as users would like. In an attempt to improve the situation, Suren Baghdasaryan has [proposed](<https://lwn.net/ml/linux-kernel/20210723011436.60960-1-surenb@google.com/>) the addition of a system call named `process_mrelease()`. 

Systems running mixed workloads, where some tasks are more important than others, are not uncommon. If the system is being run near its maximum capacity, the relatively unimportant tasks may end up using memory that is needed by the more important work, at which point it might be better if the unimportant processes went away. Such systems often run process managers that will kill off the low-priority processes in these situations; perhaps the most widespread example of this pattern is Android, which will kill background apps if the available memory is insufficient for whatever is running in the foreground. Cloud-computing systems will also kill low-priority, best-effort workloads if their memory is needed by more important work. 

Killing a process should, in principle, make its memory immediately available for other users. In the real world, though, things are not so simple. The killed process is, itself, responsible for cleaning up and freeing its resources, a task that is carried out in kernel context. If, however, the killed process finds itself blocked in an uninterruptible sleep, that cleanup work could be delayed indefinitely. There are other factors that can slow down the freeing of memory, including how busy the relevant CPU is and whether that CPU is running in a slow, low-power state. 

When this happens, the system has paid the cost of killing the process (which was presumably doing something useful) without receiving the benefits from that action. Unfortunately, those benefits tend to be needed urgently; the system would not be killing processes otherwise. Delays in process cleanup can have immediate and visible effects on the higher-priority workloads; these can include jerky response on a handset or a delay in the delivery of a cat video to an impatient viewer. 

This problem was encountered years ago in the context of the system's out-of-memory (OOM) killer, which is the kernel's last-resort response when memory runs out. Back in 2015, the [development of the OOM reaper](<https://lwn.net/Articles/668126/#reaper>) addressed this problem by taking the memory cleanup work out of the dying process's hands and making it the responsibility of a separate kernel thread. That made OOM killing significantly more robust, with the ability to free memory quickly even if the chosen process is not able to exit immediately. 

That work did not address one other unfortunate characteristic of the OOM killer, though: its opinion of what is the least important process on the system tends to differ from that of the system's users. Invoking the OOM killer may allow the system as a whole to continue functioning, but the user whose window-system server was just killed may be forgiven for not being fully enthusiastic in their celebration of that feat. 

For this reason, systems developers have tended to take the business of killing processes to rob them of their memory into their own hands. An out-of-memory handler running in user space can take more proactive steps to prevent the system from going into the OOM state to begin with, and it probably has a better idea of which processes will cause the least pain should they encounter an untimely demise. The [oomd daemon](<https://github.com/facebookincubator/oomd>) released by Facebook is one example of this kind of utility; there are others as well. 

User-space OOM killers, though, are not in the same position as the kernel's OOM killer; they must rely on the [`kill()`](<https://man7.org/linux/man-pages/man2/kill.2.html>) system call (or, more recently, [`pidfd_send_signal()`](<https://man7.org/linux/man-pages/man2/pidfd_send_signal.2.html>)) to implement the sharp end of their memory-freeing decisions. Killing a process that way does not bring the OOM reaper into play, so user-space daemons are back in the situation of having to wait for the targeted processes to release their own resources. 

Baghdasaryan's answer to this problem is a new system call: 

```
int process_mrelease(int pidfd, unsigned int flags);
```

The `pidfd` argument is a [pidfd](<https://lwn.net/Articles/794707/>) identifying the process of interest; that process must be exiting (presumably as the result of a previous `kill()` operation) when the call is made. The `flags` argument must be zero for now. This call will have the same effect as setting the OOM reaper on the indicated process, stripping away as much of its memory as possible. 

One of the reasons behind the creation of a separate call for this work is to give the system a context in which to do it. The task of going through the process's address space and freeing up all that memory will be done by the process that calls `process_mrelease()`, which may or may not be the process that killed the target in the first place. The kernel can then do this work with the priority of the calling process, and with its CPU assignments, allowing the cleanup work to be contained where it will not interfere with the (remaining) system workload. 

An alternative that was discussed with [an earlier attempt to solve this problem](<https://lwn.net/ml/linux-kernel/20190411014353.113252-1-surenb@google.com/>) was to just unconditionally reap the memory of a process when it is killed, without requiring a separate system call to make that happen. In that case, though, the work would be done in the context of the process sending the signal, which might not be welcome. A process that kills a lot of other ones — a `killall` command, for example — could be significantly slowed if that policy were to be adopted. Adding a separate system call gives user space more control over when and how that work is done. 

In [the previous posting](<https://lwn.net/ml/linux-kernel/20210623192822.3072029-1-surenb@google.com/>) of this work, the main topic of discussion was the name of the system call itself — `process_reap()` at that time. That is a reasonably clear sign that the more significant issues have been addressed and that the work may be about ready to move forward. The number of callers of `process_mrelease()` is likely to be small, but it seems there will be some situations where it will be a useful tool to have.
