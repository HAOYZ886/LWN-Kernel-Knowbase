---
title: Addressing latency problems in 2.6
url: https://lwn.net/Articles/93604/
date: "July 14, 2004"
category: Latency; Preemption; Voluntary preemption
author: 
---

The 2.6 kernel is becoming increasingly stable, and the user base is, correspondingly, becoming happier. There is, however, one remaining group of disgruntled users out there: multimedia users and developers who depend on very quick response times from the kernel. Whether you are capturing a video stream, playing a movie, or burning a disc, you need the system to respond very quickly when the hardware involved needs attention. Failure to respond in time leads to buffer overruns or underruns; those, in turn, lead to video degradation, audio skips, writable media which is suitable only for use as drink coasters or grade-school art projects, and flames on various mailing lists. 

The traffic has been growing in recent times, as it has become clear that some in the multimedia community [feel discriminated against](<https://lwn.net/Articles/93605/>): 

"We" (the audio developer community) did not participate because it was made clear that our needs were not going to be considered. We were told that the preemption patch was sufficient to provide "low latency", and that rescheduling points dotted all over the place was bad engineering (probably true). With this as the pre-rendered verdict, there's not a lot of point in dedicating time to tracking a situation that clearly is not going to work. 

The result of this discussion has been a renewed interest among the kernel developers in fixing this particular problem. It is pretty universally believed that the latency issue should be close to resolved, and that it is just a matter of fixing a few remaining trouble spots. 

One approach that has been taken is the [voluntary preemption patch](<http://lwn.net/Articles/93124/>) put together by Ingo Molnar and Arjan van de Ven. This patch tries to reduce latency by adding more scheduling points - essentially the approach that was taken back in the 2.4 days. Some things were done a little differently, however. 

The 2.6 kernel contains a hundred or so calls to `might_sleep()`. This function is a debugging aid; it is a way of marking functions which can sleep. If `might_sleep()` finds itself being called in a situation where sleeping is not allowed (while a spinlock is held, for example) it complains loudly and, hopefully, the problem gets fixed. Ingo and Arjan noted that any place which calls `might_sleep()` is, by definition, a good place to perform scheduling. So the voluntary preemption patch adds a `cond_reschedule()` call to `might_sleep()`, allowing a higher-priority process to be scheduled, should such a process exist. This tweak yields over 100 scheduling points without having to actually go into the code in that many places. 

While they were at it, Ingo and Arjan also added a few scheduling points in places that needed them, and also split up code in a couple of places which were holding locks for too long. 

This patch was not welcomed by everybody. In the mainline kernel, the `might_sleep()` call can be configured out entirely for production kernels; it is a pure debugging aid. The voluntary preemption patch turns it into a scheduler function and makes its presence required in production kernels. Some developers would rather see explicit rescheduling calls added in the places where they make sense. 

The strongest objection, however, would appear that the 2.6 kernel already implements _involuntary_ preemption via the preemptable kernel option. Any place which calls `might_sleep()` is already, by definition, preemptable, so the voluntary preemption patch adds nothing which the kernel can't already do. [Says Andrew Morton](<https://lwn.net/Articles/93607/>): 

And please let me repeat: preemption is the way in which we wish to provide low-latency. At this time, patches which sprinkle cond_resched() all over the place are unwelcome. After 2.7 forks we can look at it again. 

So why are some developers pursuing the voluntary preemption patch? At this time, very few distributors are shipping 2.6 kernels with kernel preemption turned on, mostly out of fear of creating stability problems. Kernel preemption is, itself, reasonably well debugged at this point, but it has, over the last year or so, shaken out a fair number of bugs in other parts of the kernel. Few such bugs have been found recently, but the distributors continue to take a conservative approach. Users often find bugs in surprising places, and bugs related to preemption can be incredibly difficult to reproduce and track down. The voluntary preemption patch is a way of getting some of the benefits of kernel preemption without turning on a configuration option that the distributors find scary. 

Andrew has often stated his wish to have the mainline kernel meet the needs of the distributors, so [he may eventually merge the patch](<https://lwn.net/Articles/93608/>): 

Oh I can buy the make-the-bugs-less-probable practical argument, but sheesh. If you insist on going this way we can stick the patch in after 2.7 has forked. I spose. The patch will actually slow the rate of improvement of the kernel :( 

Meanwhile, the effort to find the real latency issues is going forward. William Lee Irwin and Con Kolivas have put together [a patch](<https://lwn.net/Articles/93446/>) which tries to track down high-latency parts of the kernel. It works by making a note of when kernel code disables preemption (usually by taking a spinlock) and when preemption is turned back on again. If preemption is disabled for too long, a message is printed stating where the problem is to be found. 

ALSA users who are experiencing latency problems, and who would like to help track them down, should also be aware of the `xrun_debug` knob. It is described in [sound/alsa/ProcFile.txt](<https://lwn.net/Articles/93609/>) in the `Documentation` directory. Turning this option on causes a message and a kernel stack trace whenever an audio device suffers from a buffer overrun or underrun. This information can often be used to find the source of latency issues in short order. 

Thanks to the preempt-timing patch and `xrun_debug`, a few suspects have been turned up already. Console scrolling turns out to be one of them. ReiserFS has also come up a few times as being a source of high latency, to the point that its use in latency-critical situations is being discouraged. Ext3 has been shown to be the source of a few problems as well; the -mm tree currently contains a set of patches aimed at fixing the worst of those. Another problem can be driver `ioctl()` methods, which run with the big kernel lock held. This process is just beginning, however. 

Yet another approach can be found in [this patch](<https://lwn.net/Articles/93613/>) by Joe Korty. Software interrupts have been fingered as a potential source of latency problems; they take priority over regular kernel code, and have no real, hard limit on how long they can run. Joe's patch pushes all software interrupt handling into the `ksoftirqd` daemon, giving the scheduler a say on when they run. In this way, high-priority user processes will see lower latencies - at the expense of higher latency for the handling of software interrupts. 

Tracking down and fixing the remaining latency problems may take a little while. But enough attention is now being focused on the problem that its resolution seems pretty well assured. The complete solution, however, requires enabling kernel preemption, meaning that, for the time being, 2.6 users in search of low latency will have to build and install their own kernels.
