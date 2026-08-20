---
title: The kernel and the C library as a single project
url: https://lwn.net/Articles/417647/
date: "November 30, 2010"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet November 30, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 30, 2010

The kernel has historically been developed independently of anything that runs in user space. The well-defined kernel ABI, built around the POSIX standard, has allowed for a nearly absolute separation between the kernel and the rest of the system. Linux is nearly unique, however, in its division of kernel and user-space development. Proprietary operating systems have always been managed as a single project encompassing both user and kernel space; other free systems (the BSDs, for example) are run that way as well. Might Linux ever take a more integrated approach? 

Christopher Yeoh's [cross-memory attach patch](<https://lwn.net/Articles/405346/>) was covered here last September. He recently sent out a new version of the patch, wondering, in the process, how he could get a response other than silence. Andrew Morton [answered](<https://lwn.net/Articles/417651/>) that new system calls are increasingly hard to get into the mainline: 

We have a bit of a track record of adding cool-looking syscalls and then regretting it a few years later. Few people use them, and maybe they weren't so cool after all, and we have to maintain them for ever. Bugs (sometimes security-relevant ones) remain undiscovered for long periods because few people use (or care about) the code. 

Ingo Molnar jumped in with [a claim that the C library (libc) is the real problem](<https://lwn.net/Articles/417653/>). Getting a new feature into the kernel and, eventually, out to users takes long enough. But getting support for new system calls into the C library seems to take much longer. In the meantime, those system calls languish, unused. It _is_ possible for a suitably motivated developer to invoke an unsupported system call with `syscall()`, but that approach is fiddly, Linux-specific, and not portable across architectures (since system call numbers can change from one architecture to the next). So most real-world use of `syscall()` is probably due to kernel developers testing out new system calls. 

But, Ingo said, it doesn't have to be that way: 

If we had tools/libc/ (mapped by the kernel automagically via the vDSO), where people could add new syscall usage to actual, existing, real-life libc functions, where the improvements could thus propagate into thousands of apps immediately, without requiring any rebuild of apps or even any touching of the user-space installation, we'd probably have _much_ more lively development in this area. 

Ingo [went on](<https://lwn.net/Articles/417656/>) to describe some of the benefits that could come from a built-in libc. At the top of the list is the ability to make standard libc functions take advantage of new system calls as soon as they are available; applications would then get immediate access to the new calls. Instrumentation could be added, eventually integrating libc and kernel tracing. Perhaps something better could have been done with asynchronous I/O. And so on. He concluded by saying ""Apple and Google/Android understands that single-project mentality helps big time. We don't yet."" 

As of this writing, nobody has responded to this suggestion. Perhaps it seems too fantastical, or, perhaps, nobody is reading the cross-memory attach thread. But it is an interesting idea to ponder on. 

In the early days of Linux kernel development, the purpose was to create an implementation of a well-established standard for which a great deal of software had already been written. There was room for discussion about how a specific system call might be implemented between the C library and the kernel, but the basic nature of the task was well understood. At this point, Linux has left POSIX far behind; that standard is fully implemented and any new functionality goes beyond it. New system calls are necessarily outside of POSIX, so taking advantage of them will require user-space changes that, say, a better `open()` implementation would not. But new features are only really visible if and when libc responds by making use of them and by making them available to applications. The library most of us use (glibc) has not always been known for its quick action in that regard. 

Turning libc into an extension of the kernel itself would short out the current library middlemen. Kernel developers could connect up and make use of new system calls immediately; they would be available to applications at the same time that the kernel itself is. The two components would presumably, over time, work together better. A kernel-tied libc could also shed a lot of compatibility code which is required if it must work properly with a wide range of kernel versions. If all went well, we could have a more tightly integrated libc which offers more functionality and better performance. 

Such a move would also raise some interesting questions, naturally, starting with "which libc?" The obvious candidate would be glibc, but it's a large body of code which is not universally loved. The developers of whichever version of libc is chosen might want to have a say in the matter; they might not immediately welcome their new kernel overlords. One would hope that the ability to run the system with an alternative C library would not be compromised. Picking up the pace of libc development might bring interesting new capabilities, but there is also the ever-present possibility of introducing new regressions. Licensing could raise some issues of its own; an integrated libc would have to remain separate enough to carry a different license. 

And, one should ask, where would the process stop? Putting nethack into the kernel repository might just pass muster, but, one assumes, Emacs would encounter resistance and LibreOffice is probably out of the question. 

So a line needs to be drawn somewhere. This idea has come up in the past, and the result has been that the line has stayed where it always was: at the kernel/user-space boundary. Putting perf into the kernel repository has distorted that line somewhat, though. By most accounts, the perf experiment has been a success; perf has evolved from a rough utility to a powerful tool in a surprisingly short time. Perhaps an integrated C library would be an equally interesting experiment. Running that experiment would take a lot of work, though; until somebody shows up with a desire to _do_ that work, it will continue to be no more than a thought experiment.
