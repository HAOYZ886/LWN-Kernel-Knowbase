---
title: Averting excessive oopses
url: https://lwn.net/Articles/914878/
date: "November 18, 2022"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet November 18, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 18, 2022

Even a single kernel oops is never a good thing; it is an indication that something has gone badly wrong in the system somewhere and a straightforward recovery is not possible. But it seems that oopsing a large number of times has the potential to be even worse. To head off problems that might result from repeated oopsing, there is currently work afoot to put an upper limit on the number of times that the kernel can be allowed to oops before just giving up and rebooting. 

An oops in the kernel is the equivalent of a crash in user space. It can come about for a number of reasons, including dereferencing a stray pointer, hardware problems, or a bug detected by checks within the kernel code itself. The normal response to an oops is to output a bunch of diagnostic information to the system log and kill the process that was running when the problem occurred. 

The system as a whole, however, will continue on after an oops if at all possible. Killing the system would deprive the users of the ability to save any outstanding work and can also make problems much harder to debug than they would otherwise be. So the kernel will do its best to continue executing even when something has clearly gone badly wrong. An immediate result of that design decision is that any given system can oops more than once. Indeed, for some types of problems, multiple oopses are common and may continue until somebody gets fed up and reboots the system. 

Jann Horn recently started to wonder whether perhaps the kernel should just give up and go into a panic (which will cause a reboot) if it oopses too many times. This could be a wise course of action in general; a kernel that is oopsing frequently is clearly not in a good condition and allowing it to continue could lead to problems like data corruption. But Horn had another concern: oopsing a system enough times might be a way to exploit security problems. 

An oops, almost by definition, will leave an operation halfway completed; there is usually no way to clean up everything that might need cleaning when something has gone wrong in an unexpected place. So an oops might cause locks to be left in a held state or might lead to the failure to decrement counters that have been incremented. Counters are a particular concern; if an oops causes a counter to not be properly decremented, oopsing repeatedly might well become a way to overflow that counter, creating an exploitable situation. 

To thwart attacks of this type, Horn wrote [a patch](<https://lwn.net/ml/linux-kernel/20221107201317.324457-1-jannh@google.com/>) putting an upper limit on the number of times the system can oops before it simply calls `panic()` and reboots. The limit was set to 10,000, but can be changed with the `oops_limit` command-line parameter. 

One might well wonder whether oopsing the kernel repeatedly is a realistic way to exploit a kernel vulnerability. A kernel oops takes a bit of time, depending on a number of factors including the amount of data to be logged and the speed of the console device(s). The development community has put a vast amount of effort into optimizing many parts of the kernel, but speeding up oops handling has somehow never been a priority. To determine how long handling an oops takes, Horn wrote a special sort of benchmark: 

> In a quick single-threaded benchmark, it looks like oopsing in a vfork() child with a very short stack trace only takes ~510 microseconds per run when a graphical console is active; but switching to a text console that oopses are printed to slows it down around 87x, to ~45 milliseconds per run. 

Based on that, he concluded that it would take between eight and 12 days of constant oopsing, in the best of conditions, to overflow a 32-bit counter that was incremented once for each oops. So it is not the quickest path toward a system exploit; it is also not the most discreet: ""this is a *very* noisy and violent approach to exploiting the kernel"". While there are almost certainly systems out there that can oops continuously for over a week without anybody noticing, they are probably relatively rare. 

Even so, nobody seemed opposed to putting an upper limit on the number of oopses any given kernel can be expected to endure. Nobody even really felt the need to argue over the 10,000 number, though Linus Torvalds did [note in passing](<https://lwn.net/ml/linux-kernel/CAHk-=wjejeg+mymJQYzjc=TeMkGkcOLTgKg4FY4tz4ueYdMsGQ@mail.gmail.com/>) that he would have preferred a higher limit. Alexander "Solar Designer" Peslyak [suggested](<https://lwn.net/ml/linux-kernel/20221107211440.GA4233@openwall.com/>) that, rather than creating a separate command-line parameter, Horn could just turn the existing `panic_on_oops` boolean parameter into an integer and use that. That idea didn't get too far, though, due to the number of places in the kernel that check that parameter and use it to modify their behavior now. 

A few days later, Kees Cook posted [an updated version](<https://lwn.net/ml/linux-kernel/20221109194404.gonna.558-kees@kernel.org/>) of the patch ([since revised](<https://lwn.net/ml/linux-kernel/20221117233838.give.484-kees@kernel.org/>)), turning it into a six-part series. The behavior implemented by Horn remained unchanged, but there have been some additions, starting with a separate count to panic the system if the kernel emits too many warnings. Cook also concluded that, since the kernel was now tracking the number of oopses and warnings, that information could be provided to user space via sysfs, where it might be useful to monitoring systems. 

No opposition to this change appears to be in the offing, so chances are that this patch set will find its way into the 6.2 kernel in something close to its current form. Thereafter, no kernel will be forced to put up with the indignity of constant oopsing for too long and, perhaps, some clever exploit might even be fended off. "Don't panic" might be good advice for galactic hitchhikers, but sometimes it is the right thing for the kernel to do.
