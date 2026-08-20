---
title: "KS2011: Coming to love control groups"
url: https://lwn.net/Articles/464288/
date: "October 24, 2011"
category: Control groups
author: "By Jonathan Corbet October 24, 2011 2011 Kernel Summit coverage"
---

By **Jonathan Corbet**  
October 24, 2011

* * *

[2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) coverage 

James Bottomley somehow drew the short straw as the person who got to lead a session on "coming to love control groups" at the 2011 Kernel Summit. By the end, it became clear that love for control groups was still somewhat short of universal; indeed, not everybody fully understands how they work or what the problems are. They should, though; as James pointed out, use of control groups is reaching a point where nobody can afford to ignore them. If you don't care today, he warned kernel developers, one day you'll wake up and find them in your subsystem. 

There are two aspects to control groups. The groups themselves are just a way of grouping processes; there may be complaints about how they are implemented, but they are not hugely expensive or intrusive. That cannot always be said about the controllers which, when associated with control [![\[James Bottomley\]](https://static.lwn.net/images/conf/2011/lc-europe/JamesBottomley-sm.jpg)](<https://lwn.net/Articles/464313/>) groups, enforce resource limits and provide isolation. Thomas Gleixner was quick to chime in that one problem is that people are "creating control groups like crazy"; he booted a box recently and found over 70 control groups on it. That, he thinks, is too many. 

Lennart Poettering responded that systemd only uses control groups for process grouping, it does not need any controllers at all. The only one it will use is the CPU controller if it is present. It became clear during the session - more than once - that he wanted to talk about systemd; that made some sense since systemd is one of the biggest users of control groups, but the point of the session was really to talk about control groups in the kernel. The discussion had to be rescued from systemd-related fights more than once during this session. 

Ted Ts'o said that one of the big problems is the interactions between controllers. Each controller is tied to a specific subsystem, but they often operate in ways that affect each other. It was asked whether it would make sense to name a "control group maintainer" who would have an overall view of the situation and try to make the controllers work better together; Ingo Molnar suggested that a developer who has worked with control groups for a while should be found to do it. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Andrew Morton seemed to think that a control group maintainer was a good idea; he said that the real problem is getting people to pay attention to all the new stuff out there. Currently he is looking at the [timer slack](<https://lwn.net/Articles/463357/>) and task counter controllers and trying to figure out if they are something we want or not. He is the "resident sitting cgroup maintainer," but would really like some input from others. An "architect" who understands how the controllers interact would be very helpful. It would also be good to create a list for control group discussions; containers@lists.linux-foundation.org has existed for a while, but did not see a lot of use even before the Linux Foundation lists went down. James said he will create a new cgroups@vger.kernel.org list. 

Ted complained about the lack of widely-used applications for controllers; most of the code out there tends to be internal company code. There was agreement that having more useful utilities for control groups would be a good thing. 

With regard to the controllers themselves, Paul Turner said that Google would, based on its experience, rip apart a lot of the controllers and rework them in a better form. Alan Cox added that a lot of the existing controllers are "research projects" that should never have made it into the kernel. There were some emphatic suggestions that controllers should have zero overhead if they are turned off, even when they are built into the kernel. It was also said that controllers should default to "off" unless the user has explicitly asked for them to be turned on. 

Another problem with controllers is their interaction with namespaces. The two often need to be used together, but their interfaces are completely different and the combination is awkward. Evidently there are patches in the works to give namespaces a more cgroup-like interface. 

Linus raised a complaint about namespaces: their implementation has required changes throughout the kernel. But there is still a lot of code that has not been made namespace-aware; that code works most of the time, but can occasionally break in random places. A lot of kernel code still looks at process IDs without taking into account the PID namespace; that code could end up acting on the wrong process at times. But people rarely notice because much of this code is still unused. 

The session ended with the identification of a potential control group maintainer. The ~~victim~~ future maintainer was not in the room to defend himself, though, so his name will be withheld for the time being. 

Next: [Memory management issues](<https://lwn.net/Articles/464292/>)
