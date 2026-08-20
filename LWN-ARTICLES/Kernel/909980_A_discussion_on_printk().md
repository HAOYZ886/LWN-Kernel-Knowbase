---
title: A discussion on printk()
url: https://lwn.net/Articles/909980/
date: "October 4, 2022"
category: Kernel messages; printk
author: "By Jake Edge October 4, 2022 LPC"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
October 4, 2022

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2022-Linux_Plumbers_Conference>)

The kernel's print function, `printk()`, has been the target of numerous [improvement efforts](<https://lwn.net/Kernel/Index/#printk>) over the years for a variety of reasons. One persistent problem with `printk()` has been that its latency is unacceptably high for the realtime Linux kernel; at this point, `printk()` represents the last piece needing changes before the `RT_PREEMPT` patches can be fully merged. So there have been efforts to rework `printk()` for latency and lots of other reasons, but those have not made it into the mainline; a recent discussion at the 2022 [Linux Plumbers Conference](<https://lpc.events/>) (LPC) seems to have paved the way for new solution to land in the mainline before too long. 

[ ![\[John Ogness\]](https://static.lwn.net/images/2022/lpc-ogness-sm.png) ](<https://lwn.net/Articles/910337/>)

A [`printk()` pull request](<https://lwn.net/ml/linux-kernel/YufsTLpighCI7qSf@alley/>) for Linux 6.0 that was [rejected](<https://lwn.net/ml/linux-kernel/CAHk-%3Dwie%2BVC-R5%3DHm%3DVrg5PLrJxb1XiV67Efx-9Cr1fBKCWHTQ%40mail.gmail.com/>) by Linus Torvalds sent `RT_PREEMPT` maintainer Thomas Gleixner back to the drawing board in search of a more acceptable way to finally clean things up. That resulted in a [patch set](<https://lwn.net/ml/linux-kernel/20220910221947.171557773@linutronix.de/>) that was posted on September 11, just before the start of LPC. Torvalds [said](<https://lwn.net/ml/linux-kernel/CAHk-=wj=CBxrztv0gs273zovMRNsG=i1jg-rf3d-aAx4WLEF1w@mail.gmail.com/>) that ""it seems to have a sane model"" based on a fairly cursory review. John Ogness, who has also been working on `printk()` issues, [scheduled a birds-of-a-feather (BoF) meeting](<https://lwn.net/ml/linux-kernel/87h71cr1gb.fsf@jogness.linutronix.de/>) for September 14 to discuss the proposed changes. 

Ogness began the meeting by describing (and demonstrating) the new code. When he and Gleixner refer to a "console lock", they are not referring to the existing, global lock for all consoles, Ogness said, but instead to a new per-console lock that would be added. Consoles would also get a state structure so that their status can be tracked. It contains the owning CPU for the console, the console priority (normal, emergency, or panic), and some flags. CPUs also track which priority messages they are currently printing, so that the CPU with the highest-priority messages can actually gain access to the printing kernel thread. There is also some nested priority tracking so that if a CPU is printing at the panic level and that causes a warning, the warning should not start printing and should instead just get added to the ring buffer. 

The per-console lock can be acquired by a CPU when it wants to print by specifying the context of what it is printing. It provides a context with the CPU ID, the message priority, and some attributes describing the acquisition type, such as whether it is a friendly or hostile takeover or whether the CPU is willing to spin-wait with a timeout. In the core, there is a single simple rule that governs which CPU gets the console: whichever has the highest-priority message to output. If a warning is being printed, for example, another warning cannot wrest control of the console—its warning will just go into the ring buffer instead—but a panic priority could take control. 

#### Takeovers

Ogness described the process of a friendly takeover. In the example, CPU 0 was currently printing at priority 1 (normal), when CPU 1 wanted to print a warning at priority 2 (emergency); any printing console must check each time it outputs a byte (or at whatever granularity the hardware handles) to see if another CPU needs to print. That `console_can_proceed()` check looks to see if the priority of the other printing request is higher; if so, it relinquishes the printing to the other CPU, which will then return control to CPU 0 when it is done. 

[ ![\[Thomas Gleixner\]](https://static.lwn.net/images/2022/lpc-gleixner-sm.png) ](<https://lwn.net/Articles/910338/>)

In case CPU 0 has been preempted or cannot relinquish the console in a friendly fashion, there is a facility for a hostile takeover by the CPU with a higher priority printing task. All of the takeovers are accomplished using atomic compare-and-exchange operations. It is important to note that while there is only a single CPU actually physically printing to the console at any given time, other CPUs can still use `printk()`; the data goes to the ring buffer and will eventually get out to the console—at least that is the hope. 

Mathieu Desnoyers asked what happens with the printer thread that has been the subject of a hostile takeover. Ogness said that the `console_can_proceed()` function will recognize that something has changed in the console state, which indicates the hostile takeover, and it will error out. Gleixner added that policies could be added to determine what to do because the decision is context- and driver-dependent. 

Console drivers can mark themselves as being in an unsafe state, or just generally unsafe, so that system policies can determine when and how to use them. For example, in a panic situation, it may be that the system does not want to print on an unsafe console if there is a safe console available; perhaps the persistent store (pstore) driver should be tried before any of the others, but that decision should be made at a different level in the system, he said, not in the core of the console-handling code that is being proposed. 

Today, if there are two safe consoles and one unsafe, they will all be written to in order, possibly leading to a single line on the two safe consoles and a lockup on the third so no further data gets printed. The idea is to provide the console drivers with enough information that they can make better decisions on what to do, Ogness said. Instead of just getting a string and a length, they will know the state of things: the message priority, whether the console has been subject to a hostile takeover, and so on. "We have the tools and information to make intelligent decisions" about how best to ensure the output gets to users. 

`printk()` maintainer Petr Mladek asked what happens with any backlog of messages that are still in the ring buffer when a takeover occurs; do the older messages get printed before the new higher-priority message? Gleixner said that currently the older messages are output first. That was done, in part, because that is how things work today, but an argument could be made that a higher-priority message should preempt what's already queued up in the ring buffer. For now, it has to be that way because there will be a mix of new and old drivers; once everything gets converted, different choices could be made. 

#### Incremental

Unlike some of the earlier proposals, this is an incremental approach, Ogness said. There is a new console flag to indicate drivers that implement the mechanisms; that means they provide `write_thread()` and `write_atomic()` callbacks and follow the new protocol. `write_thread()` is called from a thread context, thus it can sleep, while `write_atomic()` is called from atomic mode so it cannot block. Console drivers will be updated and those that have not (yet) been updated will still continue to function exactly as they currently do. For testing, Gleixner and Ogness used a ""a hacked up version of the early uart8250 console"", which was [posted](<https://lwn.net/ml/linux-kernel/877d2ayh2s.ffs@tglx/>) to the thread as well. The `8250_early` console is safe for both reentrancy and non-maskable interrupt (NMI), which made for a reasonable test environment: 

> [...] we wanted to concentrate on validating the core mechanisms of friendly handover and hostile takeovers instead of dealing with the horrors of port locks or whatever at the same time. 

The hope is that the advantages of having threaded and atomic support will put some pressure on console developers to update their code, Ogness said. In the end, only systems that have all of their consoles working under the new scheme will truly benefit from it. He and Gleixner have been working with Daniel Vetter and the graphics developers to try to ensure that the graphics console drivers can be converted to work under the new mechanism. 

In answer to a question about partial writes, Ogness noted that there is a per-console sequence counter that is kept in the new state structure. That sequence counter is only updated when a record has been fully printed; if a console gets interrupted by a takeover, it will stop printing mid-record before the sequence counter gets incremented; the CPU that takes over will then pick back up at the beginning of the record. One of the demos would show that behavior, he said. 

Steven Rostedt asked about systems with both old and new consoles, which will not gain the benefit of this work; would there be a way to turn off any old-style consoles? Gleixner said that a kernel command-line option will be added to "ignore all old-style consoles". Realtime kernels will need to enforce the use of that option, he said. 

#### Demo

Ogness ran a demo in QEMU that showed the console output of the system on the hacked-up serial console, which was running at 9600bps. He wanted to exaggerate the slowness of a serial console, which continued printing boot messages long after the kernel has actually finished booting. He changed the console to 14,400bps then booted the system again; once it got to the point where threaded printing was active (and running on CPU 0, which is the boot CPU), he caused a warning on CPU 1\. Immediately, the printing switched to atomic mode on CPU 1 and continued to print the backlog of boot messages. 

Looking at the output, it showed the timestamps of the messages when they actually occurred, not when they were printed, which is a feature of the new mechanism. Roughly seven seconds after the system booted, he triggered the warning, which eventually showed up in the output once the backlog of boot messages had been printed. It showed CPU 1 requesting to print at priority 2 and CPU 0 doing a friendly handover. Then there is a 12-second gap before CPU 1 starts printing the warning because that is how long it took to flush the backlog. 

He scrolled back to the point where CPU 1 took over, which shows a partial line from CPU 0 in threaded mode, directly followed by the same line in full as printed by CPU 1 in atomic mode. That looks a little ugly, perhaps, but since it is the same driver and the state tracks what is going on, something cleaner could be done, he said. For example, a newline could be printed after the partial line, Gleixner said. There was also some discussion of having the atomic-mode write prepend some information before it starts outputting the current line (followed by the backlog) to indicate that the console takeover happened at that point. 

Gleixner also clarified that the timestamp shown is when the `printk()` message got added to the ring buffer, not when the printer actually output the message to the console. The backlog is processed before the warning is added to the ring buffer, but that could change. Perhaps it is more important to get the warning information out rather than spend time printing some giant chunk of backlogged messages. All of the shed paint colors are available, "but we are not using any of those paints yet", Gleixner said; maybe it will make sense to do so when all of the console drivers are converted. 

Ogness moved on to another demo. In this case, he caused a warning on the boot CPU (CPU 0) once it had gotten to threaded printing mode; that caused the CPU to do a hostile takeover into atomic mode. He then sent an NMI and, since the kernel was built to panic on unknown NMIs, a kernel panic ensued. That started more discussion about the printing order of messages and whether the panic messages should take precedence. 

But Torvalds noted that it is important to get the backlog out because it likely contains the context of what caused the further problems. The output from the demo did not look right because it is a made-up example, he said, but in real-world situations, the backlog provides important information and the panic messages will not make sense without them. It is important to ensure that the stack trace and other information gets into the ring buffer and that the backlog gets printed once that has been done. 

Gleixner agreed and said that there are lots of details that will need to be worked out over time. The purpose of the meeting was to determine if the approach is reasonable, if it should be pursued, or if they should "just give up and say that `printk()` is going to be a land mine forever". 

#### Console changes

Greg Kroah-Hartman said that he could not really answer that question without knowing more about what would need to change in the console drivers. Gleixner said that the drivers need to provide the two callbacks; in the case of fairly simple serial drivers like the one they used for testing, the two callbacks are quite similar. More complicated consoles, such as graphics and network consoles, will require more work to decide what can actually be done in atomic context by the `write_atomic()` callback. 

Kroah-Hartman noted that USB serial consoles require enabling interrupts, which will not work in the atomic context; what can be done about those? Gleixner agreed that they will not work in some situations, but the driver will have enough information to make an informed decision and to just return an error or mark itself as unable to be used in an atomic context. Right now, "we do try, pray, and die" for those types of consoles, but they can simply be skipped (or deferred) under the new scheme. 

Rostedt asked about trying these skipped consoles after the consoles that can be written from any context are finished. Gleixner said that needs to be an administrative choice because there is a likelihood that trying those other consoles will lock up the system. The kexec developers are concerned that the subsystem will not be reached if console printing locks up the system. It should be possible for the administrator to configure that the logs only go to pstore, for example, to try to ensure that the kexec of a new kernel happens after a crash. 

There is a concern that for some systems, such as desktops and embedded systems, the USB serial console is all that is available. The console may not work well (or at all) if it cannot use interrupts, but it _might_ get some amount of information out, which could be critical. It was agreed that there will be some way to indicate that a console of last resort is tried once other, safer possibilities have been exhausted. 

It was also agreed that the whole concept of "early consoles" should go away once the consoles have been converted. The `write_atomic()` callback will handle the early console duties. The hacked-up driver being used for testing does exactly that: it registers as an early console, then just continues running as a regular threaded console once the system reaches the point where threads are available. 

After David Woodhouse professed his love for Gleixner (and not for the first time, he said to laughter), Gleixner wanted to make something clear: 

> `printk()` is the last thing I am going to clean up in the legacy mess of Linux. Then I am going to hand over to young people who can start cleaning up the stuff I put into the kernel 20 years ago. 

With that, the meeting was basically done, though Ogness did show another demo. It would seem that there are no major objections to the direction that is being proposed. Ogness [posted a report of the meeting](<https://lwn.net/ml/linux-kernel/875yheqh6v.fsf@jogness.linutronix.de/>) as well as a [set of preparation patches](<https://lwn.net/ml/linux-kernel/20220924000454.3319186-1-john.ogness@linutronix.de/>), which are now under discussion; Mladek has [taken](<https://lwn.net/ml/linux-kernel/YzXI3ztt3kpMbFt1@alley/>) five of the cleanup patches for 6.1. It may take a while to work through the rest, but there is reason for hope that this longtime problem area is being dealt with—that will finally pave the way for the last bits of the `RT_PREEMPT` patch set to be merged. One guesses that might be cause for a bit of celebration when it happens. 

[I would like to thank LWN subscribers for supporting my travel to Dublin for Linux Plumbers Conference.]
