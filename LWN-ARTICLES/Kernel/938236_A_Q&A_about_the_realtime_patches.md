---
title: "A Q&A about the realtime patches"
url: https://lwn.net/Articles/938236/
date: "July 18, 2023"
category: Realtime
author: "By Jake Edge July 18, 2023 EOSS"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
July 18, 2023

* * *

[EOSS](<https://lwn.net/Archives/ConferenceIndex/#Embedded_Open_Source_Summit-2023>)

In a session at the 2023 Real Time Linux Summit, Thomas Gleixner answered questions about the realtime feature of the kernel, its status, and the [Real-Time Linux](<https://wiki.linuxfoundation.org/realtime/start>) project's plans for the future. The talk was billed as a "Q&A about `PREEMPT_RT`" with a caveat: "anything except `printk()` and documentation". As might be guessed, the first two questions were on just those topics, but there were plenty of other questions (and answers) too. The summit was held in conjunction with the inaugural [Embedded Open Source Summit](<https://events.linuxfoundation.org/embedded-open-source-summit/>) in Prague, Czechia at the end of June. 

#### Documentation and `printk()`

Right at the start of the session, Steven Rostedt could not resist asking: "what's wrong with documentation?" That was met with a big laugh from Gleixner; "lots", he said. The biggest problem with documentation "is that it mostly doesn't exist" for realtime Linux. His `printk()` caveat was because the usual question is "when will it be done?", but that is "subject to crystal balls". He would be happy to answer [technical questions about "why `printk()` is a horror-show"](<https://lwn.net/Articles/909980/>). 

[ ![\[Thomas Gleixner\]](https://static.lwn.net/images/2023/rtls-gleixner-sm.png) ](<https://lwn.net/Articles/938411/>)

With that, he advanced to his second (and final) slide: "Questions?", which elicited a big laugh from the audience. Tim Bird asked the next question, inevitably on the second "off-limits" topic: "is `printk()` okay if you are not using a serial console?" Gleixner said "no ... I mean, kind of"; there were some problems in the `printk()` core, aside from using consoles, that made it unsuitable to use with the realtime patches. Those have been fixed, but there are still the problems with using the console driver. Those problems are not truly realtime-specific, but running the kernel with a realtime configuration makes them even more obvious. 

The `printk()` code contains a large amount of duct tape, he said, which is a pattern that the realtime developers have encountered in multiple parts of the kernel along the way. For example, the [CPU hotplug code was in a similar position](<https://lwn.net/Articles/537562/>); everyone knew that the code was broken from a design perspective. Instead of fixing the design problems, more and more duct tape and ... other stuff ... has been added in, to the point where it "slowly composts into concrete, but it doesn't work". Eventually, "you have to bite the bullet and rewrite it". 

Bird said that he had looked at the realtime patches recently, noting that there are around 80 of them scattered around the kernel, though mostly related to the serial console, with only about 4000 lines of code. He has been telling people that most of it is now upstream and that they do not have to apply the patches; "is that correct?" Gleixner had an initial one-word answer for that: "no". 

You still cannot enable realtime on the mainline kernel due to the lack of the "`printk()` bits and pieces"; the other patches in the set are for things that can be disabled, so those are not required. Once the threaded `printk()` patches hit the mainline, then it will be time to ask Linus Torvalds to enable realtime for x86 and arm64. The problems with `printk()` have been solved, he said, according to John Ogness, who has been working on the code, and `printk()` maintainer Petr Mladek. "I will believe it once it hits upstream", Gleixner said. 

#### THP and networking

An attendee asked about transparent huge page (THP) support; currently, it is disabled when `PREEMPT_RT` is chosen. He wondered if there is something that can be done about that. Gleixner said that the problems with THP for realtime need to be fixed, "patches welcome". The realtime project has been focused on getting other things done, and has not tackled THP yet. There is no technical reason why the two cannot work together, they just do not right now. The THP migration and coalescing for memory have unacceptable latencies for the realtime kernel. 

The attendee mentioned the advantages of reduced translation lookaside buffer (TLB) pressure that come with THP, which Gleixner acknowledged, but said that the project needed to prioritize getting the core of the patch set into the mainline. There is nothing stopping others from doing that work (or hiring a consulting company to do so); the project will probably look into it at some point in the future, but it would be better if others who need the feature take it on now. 

The priority of the software interrupts was the subject of a second question from the attendee; he wondered if their priority could be increased for the realtime kernel. Gleixner said that the priorities should be set from user space by the administrator, based on the needs of the system as a whole. The problem is that "software interrupts are semantically ill-defined", so the priority that might work for one application would be totally wrong for another. Those interrupts are "context stealing and not really controllable"; the network developers have defended using software interrupts "tooth and nail for a decade", but they have come around to the idea that they need to rethink that, he said. 

The attendee said that currently networking is basically broken for realtime processes; but Gleixner said that it was a complaint "about facts that have been well-known for years". Once again, he wondered why people were simply complaining, rather than digging in and working on the problem. 

Another attendee noted that you can switch the NAPI thread to a realtime priority using sysfs. Gleixner said that the networking developers are moving to a [threaded NAPI](<https://lwn.net/Articles/833840/>) right now, which solves a lot of the problem for realtime, but not all of it. There are still lots of bottom-half disables within the networking code, but those can be removed once networking fully switches to threaded NAPI. 

He likened the local bottom-half disable (i.e. `local_bh_disable()`), which prevent software interrupts from running, to the big kernel lock (BKL). Though it is a per-CPU lock, it is completely unspecified what `local_bh_disable()` protects, just like the BKL. And, as with the BKL, removing those calls breaks things, "but you couldn't tell why". 

The process of removing the BKL was useful, in that regard, because it allowed the kernel developers to figure out what it was protecting everywhere within the kernel, with one exception: the TTY layer. That brought up a question about the TTY layer and the realtime patches. It turned out that the attendee really wanted to be able to use all of the UART devices available in the kernel, but the only path to those devices right now is via the TTY layer. 

#### Toast and TTYs

"If you go through TTY, you're toast", Gleixner said. "Good luck fixing the TTY layer", he continued; he would not be "touching that with a ten-foot pole, even if you pay me money". If there is a need for serial communication from realtime processes, then some other mechanism needs to be added because TTY "is unfixable" for realtime. 

The attendee said that they were not sending much data via the serial device, but Gleixner said that did not matter; if the only code path to access the device is via the TTY layer, then "you have a problem". If there is a real use case for non-TTY access to these devices, then some other code path could be added; the attendee agreed that his use case has nothing to do with TTY. 

In fact, the questioner said that he has been maintaining an out-of-tree UART driver since the Linux 1.2 days, but it relies on a particular chip, which may not continue to be available. Gleixner said that a problem known since 1.2, which was released nearly 30 years ago, seems like it should have been fixed long ago by working with the upstream kernel developers on a proper solution. There is already support for so many different kinds of oddball devices, adding another should not pose much of a problem, given that there is a use case for it and a reason why the TTY layer needs to be bypassed. 

A question about a return of the i915 (Intel graphics) driver brought laughter from Gleixner and much of the room. It is apparently disabled for realtime kernels. The only way to get a driver for that hardware is to wait for the new driver that is under development, Gleixner said. The current driver is not fixable and the patches that are in the realtime tree "are extremely horrible"; perhaps they could eventually be merged, but it will have to come later, and he seemed skeptical about it even then. 

Some of the locking code paths in the existing i915 driver "are completely homebrew and out of any rational locking scheme in the world". That is one of the reasons that the new driver is being developed; "the replacement driver stack is coming along, you just have to wait for it". It is another example of "train wrecks in the kernel" that the realtime developers have tried to fix along the way. 

#### Things to avoid

An attendee asked if there was a list of things to avoid using with the realtime patches beyond the TTY layer and i915 driver that had already been mentioned. Gleixner said that i915 actually works with the "hacky patches" in the realtime tree, at least "by some definition of 'works'". But using the TTY layer from a realtime task should surely be avoided; you can call `printf()` from your highest priority task, but it probably is not the best idea. Doing I/O from a realtime task is not generally the right design. 

The questioner wondered about filesystems; are they problematic when running the realtime kernel? Gleixner said that he has not seen any problems with filesystems for a long time. Daniel Bristot de Oliveira said that he had seen lengthy latencies due to Btrfs recently, but Gleixner was not aware of those reports. It is the case that other kernel developers can always "needlessly slap a `preempt_disable()` somewhere in their code"; those kinds of problems need to be tracked down and the developers have to be asked not to do that. It is part of why he is "urgently needing to find the cycles to complete the 'Kernel Developers Guide to RT'"; once that is done, the realtime developers can point other kernel hackers at it. 

But filesystems largely stay out of the way, because they are not part of the realtime computation, he said; "if you write your logfile from your realtime task, fine, you asked for it ... if the disk stalls, you wait". What about an in-memory filesystem of some sort, he was asked. Gleixner said that might work, "but, seriously, don't do it". Doing so violates all of the principles of realtime, he said; that is not a Linux-specific problem as all of the different realtime operating systems will warn people away from `write()`, `read()`, and the like. 

A realtime program should either read its data up front or, if it needs to continuously update the data, have another, non-realtime process with large buffers to do the reading, he said. If the realtime task needs to write data continuously, it should be written to a ring buffer that a separate non-realtime task writes out. That is basic realtime theory, which is not at all Linux-specific, he said. 

There are systems that need to handle streaming data in realtime, from cameras for example, an attendee said; how should something like that be done? "That's a system-design problem", Gleixner said; you will need a dedicated network queue, for example, but there is no "general recipe how to make that work". The current networking code does not work all that well with realtime, but a system can be tuned to the point where it can handle high-speed, streaming data. There are also options for handling the network traffic in user space to avoid some of the problems with realtime and the current networking code. 

#### Long journey

Bird asked about the [Intel acquisition of Linutronix](<https://lwn.net/Articles/885903/>), which employs Gleixner and some of the other realtime developers; he wondered if Intel was now funding work on realtime Linux. Gleixner said that Intel had always helped fund the realtime work via the Real-Time Linux project; both Intel and Arm have an interest in realtime Linux, which is reflected in their project membership. Kate Stewart, Linux Foundation VP for Dependable Embedded Systems and the organizer of the project, said that Intel, Arm, TI, National Instruments, Red Hat, and [others](<https://wiki.linuxfoundation.org/realtime/rtl/members>) have all been part of the "long journey" to get the full realtime patch set into the mainline. 

Rostedt noted that the long journey would be 20 years in 2024, but Gleixner said that was only the public part of the journey. It was first [posted to the Linux kernel mailing list](<https://lwn.net/Articles/105948/>) in 2004, but for him the journey started at the end of 1999. That means it will be 25 years for him since the beginning of the project; "it's a long journey and there are a lot of things we need to address and improve over time", though there is "only so much capacity". He has tried working day and night, but has found that "it doesn't make things more effective". 

The final question was about the role of the [cyclictest](<https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start>) tool; is it a good reference application? Gleixner said that cyclictest is useful for testing, but that it does not "resemble any particular real-world application". The questioner wondered if there were any good examples of real-world applications that could be reviewed. Gleixner said that he did not know of one, but that cyclictest and other test/benchmarking applications do provide a kind of basic reference implementation; however, real-world applications have a wide variety of requirements and levels of complexity. 

Part of the problem with coming up with a reference realtime application is the need for specialized hardware, Gleixner said. That is a difficulty for testing and benchmarking realtime systems, he said; the results are effectively not reproducible without access to the same hardware. It is particularly hard to integrate such a test into continuous-integration (CI) systems. There is infrastructure that allows people running CI or other tests on their hardware to report it back to the realtime project, which can help detect and find regressions and other bugs. With that, Gleixner noted that he was no longer standing between attendees and the bar (or other evening activities), and the 2023 Real Time Linux Summit was complete. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for assisting with my travel to Prague.]
