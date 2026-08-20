---
title: Disallowing perf_event_open()
url: https://lwn.net/Articles/696216/
date: "August 3, 2016"
category: "Performance monitoring; Security-Kernel hardening"
author: "By Jake Edge August 3, 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
August 3, 2016

An Android security measure that limits the ability of processes to access the perf events subsystem ran up against some, perhaps surprising, resistance when it was recently proposed for the mainline. The [patch](<https://lwn.net/Articles/696234/>) simply provides another setting for the `kernel.perf_event_paranoid` sysctl parameter to disallow unprivileged processes from accessing the `perf_event_open()` system call at all. It is currently used in both Android and Debian kernels, but some kernel developers see it as too much of a "big hammer" approach. 

Jeff Vander Stoep posted the patch on July 27\. It adds a another value that can be set for the sysctl parameter (i.e. `kernel.perf_event_paranoid=3`) that restricts `perf_event_open()` to processes with the `CAP_SYS_ADMIN` capability. Currently, `perf_event_paranoid` is set to 2 by default, which disallows access to some perf features (raw tracepoint access, CPU event access, and kernel profiling) to processes without the proper capabilities; the patch does not change the default. He also submitted another [patch](<https://lwn.net/Articles/696236/>) that would allow configuring the kernel to make 3 be the default `perf_event_paranoid` value. 

In the first patch, he noted five vulnerabilities worthy of CVE numbers that have recently been found in perf and argued that allowing access to it increases the attack surface of the kernel. For production kernels, that may not make sense, so the patch is intended to allow administrators to restrict access to perf, while still providing a means for developers and others to access the tool as needed (by granting `CAP_SYS_ADMIN`). The patches are based on the grsecurity PERF_HARDEN feature and were [first proposed](<https://lkml.org/lkml/2016/1/11/587>) by Ben Hutchings back in January. At that time, he said it had been running in the Debian kernel since August 2015 with no complaints. 

It is a fairly simple and straightforward change, but Peter Zijlstra [objected](<https://lwn.net/Articles/696240/>) that providing a way to turn off perf because of some bugs was heavy-handed: ""We have bugs we fix them, we don't kill complete infrastructure because of them."" He also thought that it would inhibit new and innovative uses for the tool: 

So the problem I have with this is that it will completely inhibit development of things like JITs that self-profile to re-compile frequently used code. 

I would much rather have an LSM hook where the security stuff can do more fine grained control of things. Allowing some apps perf usage while denying others. 

Daniel Micay [noted](<https://lwn.net/Articles/696242/>) that the functionality would still be available to privileged processes and that Android will allow access by unprivileged processes, but that capability must be enabled from the `adb` shell. Furthermore: 

It isn't even possible to disable the perf events infrastructure via kernel configuration for every architecture right now. You're forcing people to have common local privilege escalation and information leak vulnerabilities for something few people actually use. 

This patch is now a requirement for any Android devices with a security patch level above August 2016. The only thing that not merging it is going to accomplish is preventing a mainline kernel from ever being used on Android devices, unless you provide an alternative it can use for the same use case. 

Micay was skeptical that an LSM-based approach would work, as was Kees Cook, who [said](<https://lwn.net/Articles/696245/>): ""I'm not against an LSM, but I think it's needless complexity when there is already a knob for this but it just doesn't go 'high' enough."" He also noted that bugs live a long time (""an average of 5 years from introduction to fix"") and they can last even longer when you take product update lifecycles into account. He argued that administrators need the ability to reduce the attack surface of their systems: 

Being able to remove attack surface is a fundamental first step of security defense, and things like perf, user namespaces, and similar APIs, expose a lot of attack surface when they are enabled. And the evidence for this attack surface being a real-world risk is in the history of security vulnerabilities (that we know about!) in these various APIs. 

Now, obviously, these API have huge value, otherwise they wouldn't exist in the first place, and they wouldn't be built into end-user kernels if they were universally undesirable. But that's not the situation: the APIs are needed, but they lack the appropriate knobs to control their availability. 

The use case Zijlstra mentioned might be a good reason to change the value of the setting, Cook said, but there are other use cases where administrators want to be able to reduce their systems' attack surface when running a pre-built kernel. But Zijlstra [disagreed](<https://lwn.net/Articles/696247/>); he is concerned that having this knob available will mean that administrators blindly apply it. That would have the effect of stopping the development of use cases like he described: 

Having this knob will completely inhibit development of such applications. Worse it will probably render perf dead for quite a large body of developers. 

Cook was undeterred, however, [saying](<https://lwn.net/Articles/696248/>) that the feature is based on a risk analysis of the attack surface, and that there are ""hundreds of millions of end-users for whom perf is not needed"". Beyond that, though, Zijlstra's argument assumes that the knob is not available, but that simply isn't true: 

I've never suggested it be default disabled: I'm wanting to upstream the sysctl setting that is already in use on distros where the distro kernel teams have deemed this is [a] needed knob for their end-users. All of the objections you're talking about assume that the knob doesn't exist, but it does already. It's just not in upstream. 

Ingo Molnar [agreed](<https://lwn.net/Articles/696250/>) with Zijlstra that the approach was ""too coarse"". He suggested that perf is not just a ""narrow debugging mechanism"" that can simply be turned off, but that it is a ""wide scope performance measurement and event logging infrastructure that is being utilized not just by developers but by apps and runtimes as well"". 

Micay [pointed out](<https://lwn.net/Articles/696253/>) that the wide scope of perf is part of its problem from a security perspective. Because it has been a ""frequent source of vulnerabilities"", it has been disabled by some distributions. Part of the problem also lies outside of the core kernel, he said: ""It's extended by lots of vendor code to specific to platforms too, so it isn't just some core kernel code that's properly reviewed."" 

The coarseness of the setting also [concerned](<https://lwn.net/Articles/696260/>) Eric W. Biederman. He suggested that many of the features to reduce the attack surface amount to a ""system wide off switch"" for features like user namespaces and perf. The result is that new applications cannot take advantage of these features, which turns the attack-surface reduction into ""great big denial of service attacks on legitimate users"". He also suggested several ideas for ways to make the feature less coarse: ""I vote for sandboxes. Perhaps seccomp. Perhaps a per userns sysctl. Perhaps something else."" 

That's about where things stand at this point. The second patch to allow configuring the kernel to default to denying access to perf has seemingly been [dropped](<https://lwn.net/Articles/696264/>). The first will undoubtedly live on in grsecurity, Android, and Debian (at least), which seems to undermine the concern that Zijlstra, Molnar, and Biederman have—as Cook said, the change has already happened in some places. Whether a more fine-grained approach emerges remains to be seen, but it is a little hard to see who would work on it. Distributions already have their solution at this point.
