---
title: Fighting fork bombs
url: https://lwn.net/Articles/435917/
date: "March 29, 2011"
category: "Fork bombs; OOM killer; Security-Security technologies"
author: "By Jonathan Corbet March 29, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 29, 2011

Unix-like systems tend to be well hardened against attacks from outside, but more vulnerable to attacks by local users. One of the softer spots in most systems has to do with "fork bombs" - processes which madly `fork()` until they run the system out of resources. These attacks are difficult to defend against and difficult to stop without a reboot; they can also, at times, be created inadvertently. If Hiroyuki Kamezawa has his way, fork bombs will be less of a problem in the future. 

The problem with fork bombs is that they are moving targets; by the time a system administrator notices a rapidly-forking process, it may have created vast numbers of children and exited. Killing processes individually in a fork bomb situation is not really an option; even a program written especially for this task can be hard put to keep up with the stream of new processes. There is just no way to get a handle on the entire tree of offending processes from user space. So it is not surprising that the best response in this situation can be to hit the Big Red Button and start over. Even if, as in Kamezawa-san's case, hitting the button involves walking to another building where the afflicted system is housed. 

Indeed, it can be hard to get a handle on this tree from kernel space as well. The process tree only exists, as such, as long as the parent processes remain alive; once a process exits, all of its children are reparented to the init process. That causes a flattening of the tree structure and makes it hard to identify all of the processes involved in the attack. So [Kamezawa-san's patch](<https://lwn.net/Articles/435220/>) starts with the addition of a new process tracking structure. It is organized as a simple tree reflecting the actual family structure of the processes on the system. It differs from existing data structures, though, in that this "history tree" persists even when some processes exit. That allows the kernel to view the entire tree of processes involved in a fork bomb even if those which launched the attack have long since gone away. 

Keeping the entire history of all processes created over the lifetime of a Linux system would be a costly endeavor. Clearly, there comes a point where history needs to be discarded. Every so often (30 seconds by default), the kernel will try to determine whether there might possibly be a fork bomb attack in process; if no signs of an attack are detected, any tracking history which has existed for more then 30 seconds will be deleted. 

How does the kernel decide whether it might be under attack? The way fork bombs incapacitate a system is usually through memory exhaustion, so the code looks for signs of memory stress: in particular, it looks to see if there have been any memory allocation stalls or kswapd runs since the last check. It also looks at whether the total number of processes on the system has increased. If none of those checks shows any reason for concern, the older history data will be removed from the system. If, instead, memory allocations are getting harder to come by or the number of processes is growing, the tracking structure will be kept around. 

If a fork bomb runs the system out of memory, the kernel's first response will be to fire up the out-of-memory (OOM) killer. Given time, the OOM killer might manage to clean up the mess, but the fact of the matter is that the OOM killer is designed around finding the one process which is creating the problem and killing it. The OOM killer cannot identify a whole tree of rapidly-forking processes and do away with all of them. 

Enter the fork bomb killer, which is invoked by the OOM killer. The fork bomb killer will perform a depth-first traversal of the process history tree, filling in each node with information on the total number of processes below that node and the total memory used by those processes. At the end, the process with the highest score is examined; if there are at least ten processes in the history below the high scorer, it is deemed to be a fork bomb; that process and all of its descendants will be killed. Problem solved - hopefully. 

There are a couple of control knobs which have been placed under `/sys/kernel/mm/oom`. History tracking will only be performed if `mm_tracking_enabled` is set to "`enabled`" (which is the default setting). The value in `mm_tracking_reset_interval_msecs` controls how often the process tracking tree is cleaned up; the default value is 30,000 milliseconds. A possibly surprising omission is the lack of a knob controlling how many descendants a process must have before it is declared to be a fork bomb; the hardcoded value of ten seems low. 

The reception for this patch has not been entirely favorable; commenters worry about the runtime cost of maintaining the tracking structure and suggest that user-space solutions may be better. Kamezawa-san [seems resigned](<https://lwn.net/Articles/435927/>) that the patch may not go in, saying ""To go to other buildings to press reset-button is good for my health."" Other administrators, who may not be within easy walking distance of their systems, may feel their health is better served by some extra fork bomb protection, though.
