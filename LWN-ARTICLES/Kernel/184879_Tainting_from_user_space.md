---
title: Tainting from user space
url: https://lwn.net/Articles/184879/
date: "May 24, 2006"
category: Tainting
author: 
---

The kernel has long used "tainting" as a way of noting that something has happened which may affect the stability of the system. Should a kernel oops occur, the resulting kernel trace includes information on the kernel's taint status. This information can then be used by developers to ask hard questions about what was really going on. The taint flag was originally added to flag the use of binary-only kernel modules, but its use has grown since then. Events which will taint a current kernel include the forced removal of a module, loading a module without proper (or matching) version information, or running an SMP kernel with processors not designed for SMP operation. Machine check exceptions and certain kinds of memory management errors will also result in a tainted kernel. 

[A recent patch by Ted Ts'o](<http://lwn.net/Articles/184557/>) expands the taint concept in an interesting way. It adds a new file (`/proc/sys/kernel/tainted`); should user space write to that file, the kernel will be marked tainted with the new "`U`" flag. The idea, says Ted, is to flag ""when userspace is potentially doing something naughty that might compromise the kernel."" It took a few more questions before [the real truth of the matter](<http://lwn.net/Articles/184783/>) came out: 

The problem is that the Real-Time Specification for Java (RTSJ) **requires** that the JVM provide class functions which provide direct access to physical memory; all physical memory. In fact, the RTSJ compliance test explicitly checks for this; it requires that you give the compliance test the address of a few hundred megs of physical memory for the test. The absolutely hilarious bit about all of this is that the same customer who wants RTSJ compliance because of federal procurement regulations is also interested in using SELinux. 

The idea of using SELinux on a system where Java code is free to mess around with physical memory does involve a fair amount of cognitive dissonance. But The Customer Is Always Right, so Ted is making this work. [Not entirely willingly](<https://lwn.net/Articles/184881/>), though: 

In fact, I was so unhappy about being forced by the RTSJ specification to do this insane thing that I wanted to make sure that if it were ever used, it would set a TAINT flag to warn people that just about anything unsane could have happened, and the system's stability was at the mercy of the competence of Java application programmers. 

Nobody has stepped forward to say that the kernel should not be tainted in such a situation. Instead, one might almost be able to merge a patch causing the kernel to emit scary horror-movie sounds as well. 

There appears to be general agreement that this patch makes sense; certainly there are plenty of situations where user-space actions might affect the stability of the system. There was one request for a log message to be stored with the user-space taint flag so that the reason for its presence would be more clear later on. A concern was also raised that some distributions were using the "`U`" flag for other reasons (to flag the presence of "unsupported" modules), though it is not clear that this is actually happening. Collisions over the use of taint flags could indeed create confusion, so Dave Jones has suggested that any taint flags used in out-of-tree code should at least be documented with a comment in the mainline kernel. Whether any such flags exist remains to be seen, however.
