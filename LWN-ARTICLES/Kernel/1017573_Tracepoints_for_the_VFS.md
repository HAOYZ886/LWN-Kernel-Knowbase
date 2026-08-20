---
title: "Tracepoints for the VFS?"
url: https://lwn.net/Articles/1017573/
date: "April 18, 2025"
category: "Filesystems-Virtual filesystem layer; Tracing"
author: "By Jake Edge April 18, 2025 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 18, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Adding tracepoints to some kernel subsystems has been controversial—or disallowed—due to [concerns about the user-space ABI](<https://lwn.net/Articles/705270/>) that they might create. The virtual filesystem (VFS) layer has long been one of the subsystems that has not allowed any tracepoints, but that may be changing. At the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), Ted Ts'o led a discussion about whether the ABI concerns are outweighed by the utility of tracepoints for the VFS. 

[ ![\[Ted Ts'o\]](https://static.lwn.net/images/2025/lsfmb-tso-sm.png) ](<https://lwn.net/Articles/1017948/>)

Ts'o began by noting that Al Viro, who has opposed VFS tracepoints over the years, was not present, but that VFS co-maintainer Christian Brauner was in attendance to give his opinions on the matter. Historically, there have been concerns about placing tracepoints at various places in the VFS, Ts'o said, such as for system calls like [`open()`](<https://www.man7.org/linux/man-pages/man2/open.2.html>) and [`rename()`](<https://www.man7.org/linux/man-pages/man2/rename.2.html>). One concern is about tracepoints in hot paths affecting performance, but he thinks that could be worked around by keeping the tracepoints at the system-call level. Another is that the tracepoints ""might potentially constrain our implementation"" because of the user-space interface question, with the [powertop incident](<https://lwn.net/Articles/442113/>) often cited as an example of the problem. 

Today, developers and users seem to be less concerned about the ABI stability for tracepoints, he said. Many are using function tracing now to get the information they need from the VFS, which is even more implementation dependent; adding tracepoints will make things more usable and maintainable. 

Brauner said that there is no real barrier to adding VFS tracepoints in his mind; the hot-path concern is valid, but that just means being careful. The [VFS debug infrastructure](<https://lwn.net/ml/all/20250209185523.745956-1-mjguzik@gmail.com/>), which is patterned on similar functionality in the memory-management subsystem, is queued up for 6.15. Adding tracepoints makes sense in that context, he said, it helps development. He does not believe there is a need to make them stable; ""we are free to remove them, I think, we are free to move them around"". 

Ts'o said that system administrators also find tracepoints useful, not just developers. ""If we put tracepoints, let's try to keep them stable; yeah we can change them, but in the ideal we don't"". 

Chuck Lever said that he has ""been a gigantic proponent of tracepoints in the NFS subsystem"". Some kind of observability is needed, but it is easy to go overboard and stick tracepoints in every function and on every return path. Since much of the filesystem community was gathered at the summit, he wanted to discuss what the goal of the effort should be. One possibility is to only add them to error paths; function tracing already exists and is easy to use for the arguments and return values. Determining which error return was taken would be a good use for tracepoints. In general, having a use case for a tracepoint, rather than just scattering them in the code, is important, he said. 

David Howells cautioned that there is a need to ensure that new tracepoints do not themselves trigger tracepoints in, say, tracefs. Accessing the tracing information should not cause more tracepoints to fire. Along those lines, he suggested that some kind of filtering was needed to isolate the tracepoints for a particular filesystem. Tracepoints for the mount path should be fine, since those operations are not that frequent, but the kernel does a lot of reads and writes. 

Brauner wondered if tracepoints for read and write were even all that interesting, but Howells said that the page flags of the buffers are. Brauner said that much of the filtering requested can already be done using bpftrace, kretprobes, and other kernel facilities. Ts'o said that the ext4 tracepoints have the `dev_t` of the filesystem being operated on as a parameter, so he can filter for a specific filesystem based on that value. 

Tracepoints are more useful than other possibilities because those often depend on functions not getting inlined by the compiler or by a [post-compilation optimization tool, such as BOLT](<https://lwn.net/Articles/993828/>), an attendee said. ""We need way more tracepoints"", he said. Brauner said that they will also be useful in the mount path, since there are so many ways to get an `EINVAL` return code; he has a friend with a ""script called 'why-did-mount-fail'"", but there are so many inlined functions that it can be difficult to determine. 

Adding VFS tracepoints seems non-controversial, someone said, to general agreement. The specifics of the tracepoints and where they are placed may be controversial, however. Brauner said that times have changed in the almost 15 years since the powertop problem and the ""observability game"" has changed as well. 

Mathieu Desnoyers, who was the original developer of tracepoints back in 2008, noted that there was another concern expressed when the question of adding tracepoints was raised in the past: tracepoints can be misused as an execution-hijacking mechanism. For example, a rootkit could potentially use a tracepoint to alter the behavior of a system call in a running kernel. Several people noted that there are other kernel mechanisms that could be used for that purpose, however, without the need for any tracepoints. It does not seem like something to worry about at this point. 

Those in the room certainly seemed to be in favor of adding VFS tracepoints and no real barriers to doing so were raised. One would guess that patches to start adding them will be posted before long.
