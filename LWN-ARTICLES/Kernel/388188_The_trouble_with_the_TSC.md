---
title: The trouble with the TSC
url: https://lwn.net/Articles/388188/
date: "May 19, 2010"
category: Timers
author: "By Jake Edge May 19, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 19, 2010

The time stamp counter (TSC) provided by x86 processors is a high-resolution counter that can be read with a single instruction (RDTSC), which makes it a tempting target for applications that need fine-grained timestamps. Unfortunately, it is also rather unreliable, so the kernel jumps through hoops to decide whether to use it and to try to detect when it goes awry. An effort to export the kernel's knowledge about the reliability of the TSC has met strong resistance for a number of reasons, but the biggest is that the kernel developers don't think that applications should be accessing the counter directly. 

Dan Magenheimer and Venkatesh Pallipadi [proposed](<https://lwn.net/Articles/388263/>) adding a `/sys/devices/tsc` directory with several entries corresponding to the kernel's internal TSC information, including the `tsc_unstable` flag, which governs whether the kernel uses the counter as a stable time source. Andi Kleen [questioned](<https://lwn.net/Articles/388278/>) the idea: 

Is this really a good idea? It will encourage the applications to use RDTSC directly, but there are all kinds of constraints on that. Even the kernel has a hard time with them, how likely is it that applications will get all that right? 

That is exactly what the patch is meant to do, Magenheimer [said](<https://lwn.net/Articles/388280/>), because applications have no reliable way to determine whether the standard system calls will be "fast" or "slow": 

The problem is from an app point-of-view there is no vsyscall. There are two syscalls: gettimeofday and clock_gettime. Sometimes, if it gets lucky, they turn out to be very fast and sometimes it doesn't get lucky and they are VERY slow (resulting in a performance hit of 10% or more), depending on a number of factors completely out of the control of the app and even undetectable to the app. 

Note also that even vsyscall with TSC as the clocksource will still be significantly slower than rdtsc, especially in the common case where a timestamp is directly stored and the delta between two timestamps is later evaluated; in the vsyscall case, each timestamp is a function call and a convert to nsec but in the TSC case, each timestamp is a single instruction. 

Depending on the hardware, `gettimeofday()` and `clock_gettime()` may be implemented as vsyscalls--virtual system calls--rather than standard system calls, which eliminates the user space to kernel transition. Vsyscalls are code that is stored in a special memory region in user space (the vdso region) that may access kernel-maintained data, like clock ticks. Using vsyscalls, the calls are (relatively) fast, but on some hardware (or virtual machines) that requires kernel-space operations to get to a reliable counter, a vsyscall cannot be used, so the calls are slower. For applications that ""need to obtain timestamp data tens or hundreds of thousands of times per second"", the difference is significant. 

But Magenheimer believes that if the kernel finds the TSC stable enough for its own timekeeping purposes, then that guarantees that it is usable by applications. Arjan van de Ven and Thomas Gleixner are quick to correct that misunderstanding. Van de Ven [notes](<https://lwn.net/Articles/388284/>) that the stability of the TSC can change under certain circumstances and there would be no way to notify the applications. His advice: ""friends don't let friends use rdtsc in application code"". 

Gleixner [goes into some detail](<https://lwn.net/Articles/388286/>) about how the TSC can get out of whack, including system management mode interrupts (SMIs) fiddling with the TSC to hide their presence, that multiple cores can have different values because of boot offsets and/or hotplugging, and that multiple sockets can introduce differences due to separate clocks or drift in the clock signals due to temperature. There is, in short, nothing reliable about the TSC: ""the stupid hardware is not reliable whether it has some 'I claim to be reliable tag' on it or not"". Gleixner did offer a possible alternative, though: 

[...] but as long as we do not have some really reliable hardware I'm going to NACK any exposure of the gory details to user space simply because I have to deal with the fallout of this. 

What we can talk about is a vget_tsc_raw() interface along with a vconvert_tsc_delta() interface, where vget_tsc_raw() returns you an nasty error code for everything which is not usable. 

Currently, there are unnamed "enterprise applications" that attempt to figure out whether they can use the TSC, and do so if they think it will work because of the uncertainty in the performance of `gettimeofday()` and friends. Magenheimer [suggests](<https://lwn.net/Articles/388294/>) that perhaps that information could be made available: 

But the kernel doesn't expose a "gettimeofday performance sucks" flag either. If it did (or in the case of the patch, if tsc_reliable is zero) the application could at least choose to turn off the 10000-100000 timestamps/second and log a message saying "you are running on old hardware so you get fewer features". 

Magenheimer also wonders if the kernel developers are suffering from "hot stove" syndrome, in that they have been burned in the past and are reluctant to even consider changes. But Gleixner and van de Ven both point out that there is no hardware that can make the guarantees that Magenheimer wants. And Gleixner has the [burn marks](<https://lwn.net/Articles/388295/>) to prove it: 

I'm unfortunately forced to deal with the 500+ different variants of borked timers and that makes me very reluctant to believe anything what chip/board/bios vendors promise. It's not the one time hot stove experience, it's the constant exposure to the never ending supply of hot stoves, which makes me nervous. 

While the discussion had various interesting analogies including hanging ropes/knives and condoms versus abstention, it did not (yet) find a car analogy. It did, however, seem to find some common ground that information about whether the clock calls are implemented as vsyscalls or system calls should be exported. That is unlikely to satisfy those that have been ""using vsyscalls for a while and still have a performance headache"", who Magenheimer [quotes](<https://lwn.net/Articles/388299/>), but there is nothing stopping applications from reading the TSC directly. Those applications just have to be prepared to handle any strange TSC behavior they encounter. 

Ingo Molnar [tries to clarify](<https://lwn.net/Articles/388301/>) the reasons that the kernel can't export the reliability information: ""The point is for the kernel to not be complicit in practices that are technically not reliable. [...] So the kernel wont 'signal' that something is safe to use if it is not safe to use."" But he also sees [some reason to hope](<https://lwn.net/Articles/388300/>): 

You could win the argument by coming up with a patch that changes gettimeofday to make use of the TSC in a reliable manner. 

I really mean it - and it might be possible - but we have not found it yet. 

Peter Zijlstra has [another solution](<https://lwn.net/Articles/388303/>) to the problem. He would like to see the kernel move to eventually disable RDTSC from user space. By emulating the instruction and logging all uses of it (and the related RDTSCP), user-space programs that use it could be identified and changed: 

Once we get most of userspace running fine, we can switch it to generating faults. 

Of course closed source stuff will have to deal with it themselves, but who cares about that anyway ;-) 

Exporting the information about whether `gettimeofday()` is "slow" or not seems like a reasonable starting point. No patches to do that have emerged yet, but it is a fairly straightforward thing to do. Eventually, something like Gleixner's `vget_tsc_raw()` may also come about, though it won't satisfy those who are unhappy with the current vsyscall performance. Those applications will just have to read the TSC themselves and deal with whatever the hardware throws at them.
