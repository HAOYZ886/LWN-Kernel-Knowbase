---
title: "KS2010: Deadline scheduling"
url: https://lwn.net/Articles/412745/
date: "November 2, 2010"
category: "Realtime-Deadline scheduling; Scheduler-Deadline scheduling"
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

Dario Faggioli is the author of the [deadline scheduling](<https://lwn.net/Articles/378044/>) patch; he made his Kernel Summit debut to introduce this work and to get an idea of whether it might make it into a future kernel. The message he got was mixed, but it was about as positive as could realistically be expected for this kind of significant new feature. 

Deadline scheduling does away with the classic notion of process priorities. Instead, each process requests scheduling of a maximum amount of CPU time within a specific deadline. The scheduler can then either arrange things to ensure that the deadline will be met or reject the request if the CPU would be overcommitted. Dario chose the case of a video player application as his main example for how this can be useful; with a deadline for each frame, the player can produce skip-free video even in the presence of significant contention for the CPU. Deadline scheduling can thus make a number of problems go away. 

Linus wasn't convinced. He has never been entirely impressed by the realtime work (he pronounced that "realtime is bullshit" in the session) and does not see deadline scheduling as the right answer to this problem. He is more optimistic about group scheduling, and especially about the recently-posted [per-tty task groups patch](<https://lwn.net/Articles/410517/>). Peter Zijlstra defended deadline scheduling, citing its ability to reject tasks which would overcommit the CPU, but Linus wasn't going for it. He says that multimedia people don't want the video player to be rejected if the scheduler can't guarantee the deadlines; they want a best-effort attempt to play the video. 

Beyond that, he says, the problem with tasks like video playback is almost never the CPU scheduler. Video skips tend to be caused by I/O problems; the latencies appear in the I/O scheduler or in the virtual memory subsystem somewhere. Messing with the CPU scheduler is the wrong approach. 

Peter said that, regardless, deadline scheduling is a feature that a number of people want. Should they press on with the work, or is it hopeless? Linus grumbled some more about how too much time goes into CPU scheduling and not enough into the other parts of the problem. But, he said, if the scheduler people want deadline scheduling in the end, he'll pull the patch. 

Linus complained that, in practice, it's impossible to set the deadlines correctly. In the end, application developers have to request the absolute worst-case execution time, even though the application will almost never need that much CPU time. That's a problem because users often want to overcommit the resources - including the processor - on their systems. In practice, it almost always works because the worst-case CPU time is not actually needed. 

Peter replied that deadline scheduling is safer because it can be made available to unprivileged applications. The maximum amount of CPU time can be bounded, as can the worst-case execution time. The scheduler can deny requests which would overcommit the amount of CPU time available. Linus replied that, on a server with multiple users, that approach still is not safe; there is no way to keep users from interfering with each other. The truth of the matter is that multi-user servers are probably not a place where using deadline scheduling will make sense. 

In general, Linus said, deadline scheduling make him nervous. He has seen attempts to fix things with scheduler tweaks before; he's afraid that they can end up making things worse. The scenario of using deadline scheduling on desktop systems is, he says, not realistic; the real problems are elsewhere. Deadline scheduling will require system tuning, and that just doesn't work on desktop systems. There is no way to tune for everybody, and desktop users tend to be uninterested in and incapable of tuning their systems themselves. 

Ted Ts'o said that video playback is an easy example to illustrate deadline scheduling, but, perhaps, it's not the best use case. This theme was to return a couple of times in the session; video was almost certainly not the best example to choose for this particular crowd. He asked for an alternative use case - something which cannot be fixed with changes elsewhere in the system. Thomas Gleixner said that OS X is using deadline scheduling for desktop tasks and it works great. But the desktop example was used because it is easy to understand; there are also a lot of industrial applications which benefit from a non-priority-based scheduling mechanism. Deadline scheduling, he says, is all about describing the work that must be done instead of tweaking priorities. 

Tim Bird said that, in his job, he's spent 18 months tweaking embedded systems for proper performance. CPU scheduling, he says, is never the real issue; the hard problems are elsewhere. So, he said, they are unlikely to look at deadline scheduling in the next ten years. 

Ted closed the session with the note that it's important to better identify the users for deadline scheduling. Without that, it will be hard to know whether the associated ABI is right. 

[Next: Regressions](<https://lwn.net/Articles/412746/>).
