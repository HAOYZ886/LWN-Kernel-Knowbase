---
title: "Seccomp: replacing security modules?"
url: https://lwn.net/Articles/443099/
date: "May 16, 2011"
category: "Security-seccomp"
author: "By Jonathan Corbet May 16, 2011"
---

By **Jonathan Corbet**  
May 16, 2011

LWN recently [discussed an enhancement to the seccomp mechanism](<https://lwn.net/Articles/441232/>) which would allow applications to restrict their future access to system calls. By blocking off some calls altogether and by using simple, ftrace-style filters to restrict the possible arguments to other system calls, a process could construct a small sandbox which would constrain it (and its children) going forward. Getting support for new security mechanisms in the kernel is often a challenge, but not this time - almost all of the reviews are positive. The biggest complaint seems to be that the patches are not ambitious enough; at least one developer would like to see developer Will Drewry shoot for replacing the Linux security module (LSM) mechanism altogether. 

Ingo Molnar has been a supporter of this work; indeed, he suggested some of the ideas which led to the current set of patches. But he is now [asking Will](<https://lwn.net/Articles/443106/>) to be a bit more ambitious in his goals. Rather than act as a gatekeeper for system calls, why not implement the ability to hook into arbitrary kernel events and filter access there? Those who have watched Ingo over the last couple of years are unlikely to be surprised to see that he suggests hooking into the perf events subsystem for this task. Perf already allows an application to attach to events to get notifications and counts; adding per-process filter expressions, he suggests, is a natural evolution of that capability. 

In other words, Ingo suggests dropping the current interface, which is implemented with `prctl()`, in favor of a perf-based (or, at least, perf-like) interface which could operate on kernel events. In principle, any software event that perf can deal with now (including tracepoints) could be used, but these events would have to be explicitly modified by kernel developers to enable this sort of "active" use. For events modified in this way, filters written in an expanded language could be provided by an application. See [this message from Ingo](<https://lwn.net/Articles/443110/>) for an example of how this sort of functionality might be used. 

One of the biggest advantages of hooking to arbitrary events is that filters could be applied at the real decision points. A filter which allows access to the `open()` system call based on the name of the file being opened is not truly secure; the application could change the name between the pre-`open()` check and when `open()` actually uses it. Checking at a tracepoint placed more deeply within the VFS lookup code, instead, would block this sort of attack. A check placed in the right location could also be more efficient, replacing several checks at the system call level. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

According to Ingo, there are a lot of advantages to providing this sort of capability. It would allow, for the first time, security policies to be set up by unprivileged applications; developers could thus take a more active role in ensuring the security of their code. The feature could be made stackable, allowing multiple application layers to add restrictions. In fact, he thinks it's such a great idea that he said: 

I argue that this is the LSM and audit subsystems designed right: in the long run it could allow everything that LSM does at the moment - and so much more. 

Someday, he said, event-based filters could simply replace LSM which he [blamed](<https://lwn.net/Articles/443111/>) for a number of ills, including stalled security efforts, desktop annoyances, infighting, fragmentation, and ""probably *less* Linux security"". Merging the code in its current form, he said, would take away the incentive to go all the way, so he'd like to see it reworked along these lines first. 

Needless to say, this idea is not universally popular in the Linux security module community. James Morris [supports](<https://lwn.net/Articles/443113/>) the merging of the current patch, which, he says, is a good way to reduce the attack surface of the system call interface, but, he said, it is the wrong place for more serious security checks. Real security policies, he said, should be done at the LSM level. Eric Paris [suggested](<https://lwn.net/Articles/443114/>) that the filter capability should be implemented as an LSM, but he also [pointed out](<https://lwn.net/Articles/443116/>) a key weakness of that approach: 

The existence of the LSM and the fact that there exists multiple security modules that may or may not be enabled really leads application developers to be unable to rely on LSM for security. If linux had a single security model which everyone could rely on we wouldn't really have as big of an issue but that's not possible. 

Getting application developers to make use of a Linux-specific security mechanism is already asking a lot. Getting them to use a mechanism which may or may not be present even on Linux systems is even harder; that may be part of why application developers have never really stepped forward to provide SELinux policies for their code. The filtering capability envisioned by Ingo would be part of the core kernel itself; that alone could help to make it the "single security model" that Eric was wishing for. 

Any such outcome is to be found well in the future, though; there are numerous obstacles to overcome. The amount of work needed to implement this capability is not trivial. Individual tracepoints within the kernel would have to be evaluated to determine whether making them "active" makes any sense. Without a great deal of care, allowing applications to block operations within the kernel could well introduce security problems of its own. Based on past experience, the developers of the existing security mechanisms in the kernel might oppose the addition of an entirely new security-related framework. Even Linus, in the past, has been resistant to the idea of creating a single security policy mechanism for the kernel. 

For the near future, Will has [indicated](<https://lwn.net/Articles/443118/>) that he will look at implementing the feature along the lines suggested by Ingo. Once some code is out there, developers will be able to see its implications and the debate can start for real. The chances of the discussion going on for some time are fairly good.
