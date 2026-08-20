---
title: The real realtime preemption end game
url: https://lwn.net/Articles/951337/
date: "November 16, 2023"
category: Kernel messages; Realtime
author: "By Jonathan Corbet November 16, 2023 LPC"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 16, 2023

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Plumbers_Conference>)

The addition of realtime support to Linux is a long story; it first [shows up in LWN](<https://lwn.net/Articles/106010/>) in 2004. For much of that time, it has seemed like only a little more work was needed to get across the finish line; thus we ran headlines like [the realtime preemption endgame](<https://lwn.net/Articles/345076/>) — in 2009. At the [2023 Linux Plumbers Conference](<https://lpc.events/>), Thomas Gleixner informed the group that, now, the end truly is near. There is really only one big problem left to be solved before all of that work can land in the mainline. 

[![\[Thomas Gleixner\]](https://static.lwn.net/images/conf/2023/lpc/ThomasGleixner-sm.png)](<https://lwn.net/Articles/951340/>) The point of realtime preemption is to ensure that the highest-priority process will always be able to run with a minimum (and predictable) delay. To that end, it makes the kernel preemptible in as many situations as possible, with the exceptions being tightly limited in scope. The basic mechanics of how that works have been established for a long time, but there have been a lot of details to resolve along the way. The realtime preemption work has resulted in the rewriting of much of the core kernel over the years, with benefits that extend far beyond the realtime use case. 

Gleixner started by noting that, while the realtime preemption project has been underway for nearly 20 years, it is actually closer to 25 years for him — he started working on realtime support for Linux in 1999\. Once it's done, he said, there will be "a big party". Is that point at hand? The answer, he said, is "yes — kind of". There is one last holdout to be dealt with: `printk()`. 

Whenever code in the kernel needs to send something to the system consoles and logs, it calls `printk()` or one of the numerous functions built on top of it. One might not think that printing a message would be a challenging task, but it is. A call to `printk()` can come from any context, including in non-maskable-interrupt handlers or other `printk()` calls. The information being printed may be crucial, especially in the case of a system crash, so `printk()` calls have to work regardless of the context. As a result, there are a lot of concurrency and locking issues, and lots of driver-related complications. 

`printk()`, Gleixner said, is fully synchronous in current kernels; a call will not return until the message has been sent to all of the configured destinations. That is "stupid"; much of what is printed is simply noise, especially during the boot process, and there is no point to waiting for it all to go out. Beyond being pointless, that waiting introduces latency, which runs counter to the goals of the realtime work, so the realtime developers have long since moved `printk()` output into separate threads, making it asynchronous. That code is a bunch of hacks rather than a real solution, though. A better job must be done to make this work useful for the rest of the kernel. 

The `printk()` problem has been worked on seriously since 2018, resulting in about 300 patches that have either gone upstream or are waiting in linux-next; this work has been [covered here](<https://lwn.net/Articles/780556/>) at times. There are, he said, three final patch sets currently in the works to finish the job. A few tricky details are still being worked on. One of those is the handover mechanism; if the kernel has an emergency message to put out (it's crashing, for example), it may need to grab control of a console that is currently printing a lower-priority message. Doing that safely from any context is not an easy thing to do. 

Another ongoing task is marking console drivers that are not safe to use in some contexts; if, for example, outputting a message during a non-maskable interrupt requires doing video-mode setting, it's just not going to work. 

Gleixner finished the prepared part of his talk by saying that, even though it's getting close, nobody should ask him when the work will be done. `printk()` is unpredictable, and he is no longer willing to even try. Even so, he expressed hopes that the rest of the realtime preemption code would be in mainline before the 20th anniversary comes late in 2024. 

[![\[John Ogness\]](https://static.lwn.net/images/conf/2023/lpc/JohnOgness-sm.png)](<https://lwn.net/Articles/951341/>) An audience member asked whether there had been any interesting changes in the `printk()` code over the last year; Gleixner answered that there have been no fundamental conceptual changes. John Ogness, who has done much of the `printk()` work, said that the handover code has been reduced somewhat, but that some work remains; there are 76 console drivers in the kernel that need to be fixed, and it may take a while until they are all done. The handover code has been changed to allow drivers to be updated one at a time rather than requiring that this work all be done at once. (See [this article](<https://lwn.net/Articles/909980/>) for more discussion on the recent `printk()` work). 

Masami Hiramatsu asked which kernel messages need to be printed synchronously; Gleixner answered that almost everything should be made asynchronous. Beyond reducing latency associated with `printk()` calls, asynchronous output allows the creation of a separate kernel thread for each console, letting the faster consoles go at full speed rather than waiting for the slowest one. He also said that the code has been changed to ensure that important messages are fully copied into the message buffer before the first line is output, just in case a faulty console driver brings the whole system down in flames. Further safety is obtained by writing to the known-safe consoles first. If, for example, there is a persistent-memory store available, messages are put there before being sent to physical devices, once again preserving the output even if a faulty driver kills the system. 

As the session closed, Clark Williams asked whether, once the `printk()` patches go upstream, Gleixner would try to push the rest of the realtime code (which wasn't discussed in this session) in the same merge window. The answer was a qualified "yes"; he might try if all of the code is staged in linux-next and seems ready to go. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event.]
