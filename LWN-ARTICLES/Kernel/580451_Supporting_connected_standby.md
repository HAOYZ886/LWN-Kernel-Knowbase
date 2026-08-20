---
title: Supporting connected standby
url: https://lwn.net/Articles/580451/
date: "January 15, 2014"
category: Power management; Software suspend
author: "By Jonathan Corbet January 15, 2014 linux.conf.au 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 15, 2014

* * *

[linux.conf.au 2014](<https://lwn.net/Archives/ConferenceByYear/#2014-linux.conf.au>)

One of the first questions an experienced Linux user asks when evaluating a new laptop is: does suspend-to-RAM work properly? Over the years, suspend support has often been an area of difficulty for Linux users, but things have slowly gotten better thanks to hard work by a number of developers. Thus, one might conclude that few users will welcome the news that all of that work may soon go out the window as the result of hardware changes currently in the pipeline. Matthew Garrett took some time during the [linux.conf.au 2014](<http://linux.conf.au/>) Kernel miniconf to describe the upcoming "connected standby" mode and how the kernel might change to support it. 

Ten years ago, Matthew said, the "horrific" advanced power management (APM) mechanism, which Linux developers had finally gotten to work reasonably well, was pushed aside in favor of ACPI. Supporting ACPI properly required a fair amount of time and effort, but, once again, kernel developers have managed to get it working, so, naturally, it's time for something else to come along. That something else is connected standby, which moves away from an explicit sleep state toward something that is part of the system idle state. 

User expectations have changed, Matthew said, and it's often undesirable for a system to go into a hard sleep state at any time. Even when the user is elsewhere and power consumption should be minimized, machines still need [![\[Matthew Garrett\]](https://static.lwn.net/images/conf/2014/lca/MatthewGarrett-kmc-sm.jpg)](<https://lwn.net/Articles/580457/>) to wake up, download email, respond to push notifications, and so on. In other words, the machine needs to always be running in some sense and able to respond to the world. Hardware manufacturers are responding by making it possible for idle systems to draw almost no power at all. They expect this mode to be used, as can be seen by the fact that machines that do not support the ACPI "S3" sleep state (a.k.a. suspend) at all will start shipping soon. 

In theory, supporting these systems under Linux should be relatively easy; we just have to make sure that they get into (and stay in) a sufficiently idle state. But that, of course, is where things get difficult; anything that brings the system out of idle will consume power and defeat the purpose of the whole exercise. And, Matthew said, we still have a number of places where the system is indeed being awakened unnecessarily. 

Within the kernel, he said, the read-copy-update (RCU) subsystem often seems to be doing mysterious things at strange times. But the real problems lie in user space, which still displays "often dreadful" wakeup behavior. A lot of problems got fixed when powertop became available, but others remain and, more importantly, user-space developers keep adding new code that wakes the system unnecessarily. And, unfortunately, there are way more user-space developers than kernel developers out there. As zombie movies have shown us over and over again, Matthew said, superior numbers will always triumph in the end. 

Since we can't count on help from user space to make connected standby work properly, some other approach will be required. One brute-force solution would be to use the process freezer to just stop user space when the system wants to go into the suspend-like idle state. That would work, but it has a significant shortcoming: a frozen user space can't listen for or respond to events. If (important) things can't happen while the system is in the connected standby state, we have, once again, lost one of the advantages that connected standby is supposed to bring. 

Perhaps that problem could be worked around by creating a special listener process that would remain runnable. That process would watch for events of interest, then wake other parts of the system when something comes in. But that solution, Matthew said, "sounds awful." 

An alternative might be to make use of the kernel's [timer slack](<https://lwn.net/Articles/369549/>) mechanism. Timers at any level of the system are always allowed to expire later than the requested time; any number of events could cause a delay to happen. Timer slack is an explicit, intentional delay added to a timer to cause its expiration to coincide with the expiration of other timers in the system. It is thus a power-saving feature: having multiple timers go off at the same time requires fewer system wakeups than having each timer expire by itself. 

Normally, timer-slack limits are measured in milliseconds at most; the connected standby case, instead, would take timer slack to a bit of an extreme. When the system is to go into the suspended state, the timer slack for most or all processes in the system is set to an infinite value. That means that no timer will ever wake the system from the suspended state; nothing will happen until some other kind of wakeup event (an interrupt, for example) comes along. Matthew said that he has played with this idea a bit; for the most part, things work better than one might expect with an infinite timer slack value. So this might be a viable path toward a solution. 

That said, there are a few things that need to be worked out. A few timers turn out to be important and should not be delayed indefinitely. So it might make sense to limit the application of infinite timer slack to a subset of the processes in the system. One possible way to do that would be to add a control group controller for setting timer slack; this idea has [come up in the past](<https://lwn.net/Articles/463357/>) for other reasons, but has not been received well by the core kernel development community. Those developers see unwanted wakeups as a user-space problem that should be fixed there but, as Matthew dryly noted, they did not volunteer to actually do that work. 

Your editor, feeling the need to play the devil's advocate, noted that the kernel contains an [opportunistic suspend mechanism](<https://lwn.net/Articles/416690/>) used to implement the Android wakelocks concept. Wakelocks were designed to solve a very similar problem: allowing the system to suspend itself in the face of poor application behavior. So why can't the wakelock mechanism be used here? Matthew's response was that wakelocks require that user-space programs be written with their use in mind; resources in an Android system often require that their use be tied to a wakelock. Classic Linux applications, instead, expect resources to be available all the time; they would have to be rewritten to work with wakelocks. Since, as has already been noted, rewriting all of user space is not really an option, wakelocks will not work in this situation. 

So, despite the loose ends in need of tying down, timer slack still seems like the best solution to the problem. That said, Matthew would be delighted if somebody were to come up with a better idea. No such better ideas were on offer at the miniconf, so we are likely to see a push toward infinite timer slack in the relatively near future. Any opposition to a mechanism for system-wide timer-slack control may yet fade away when it becomes clear that there is no other viable way to make new systems suspend properly. 

[Your editor would like to thank linux.conf.au for funding his travel to Perth].
