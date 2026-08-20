---
title: "Kernel Summit: Virtualization support"
url: https://lwn.net/Articles/94571/
date: "July 21, 2004"
category: "User-mode Linux; Virtualization"
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
Virtualization is an increasingly hot topic, despite the fact that people often do not seem to understand what it really means. For the purpose of this discussion, "virtualization" means running Linux (or another operating system) on a virtual computer which has been somehow isolated from the real, underlying hardware. Chris Wright led a session on this topic. 

There are plenty of projects out there. [User-mode Linux](<http://user-mode-linux.sourceforge.net/>) is a classic example; interest in UML seems to be picking up again in recent times. The folks working on [Xen](<http://www.cl.cam.ac.uk/Research/SRG/netos/xen/>) have created an x86 virtual machine which is capable of running a few systems; there is said to be a Xen patch getting ready for submission to the kernel which has lost many of its more intrusive parts. The [Linux VServer project](<http://www.linux-vserver.org/>) takes a different approach, seeking to isolate processes on a Linux system using `chroot()` and other traditional mechanisms. Then, there's IBM, which has been running virtual machines for a number of decades now. 

One feature often requested by virtualization projects is tickless timing. A system running one instance of the operating system can handle a timer interrupt every 1ms or so; if you are running multiple operating systems, however, the overhead gets overwhelming. 

Jeff Dike quickly went over his plans for UML. They include a move to "self ptracing" if the kernel support for it can be added; this mechanism would allow UML to run entirely within a single process, rather than the two it uses now (for its [SKAS mode](<http://user-mode-linux.sourceforge.net/skas.html>)). Jeff would also like to see `[/dev/anon](<http://lwn.net/Articles/66098/>)`, which is really a mechanism for telling the kernel not to to keep copies of specific file pages in the page cache. He has requested a configuration option which would quickly select out any devices which access actual hardware, leaving only the virtual devices. Jeff also is interested in using asynchronous I/O within UML; his goal is to never have a UML process sleep involuntarily, and AIO would help in that regard. 

While interest in virtualization is high, it is not clear what the kernel developers can do to help. The variety of approaches out there shows that the best way to do virtualization has not yet been found. Things remain very much in the exploratory stage. 

[>> Next: Security](<https://lwn.net/Articles/94572/>).
