---
title: "Kernel Summit: The customer panel"
url: https://lwn.net/Articles/94384/
date: "July 21, 2004"
category: Customer panel; Goldman Sachs
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
The second day of the 2004 Kernel Summit began with the "customer panel," being a set of presentations from large commercial Linux users on their needs and wishes. The number of presenters was smaller than planned; a couple of people found themselves unable to attend. The discussion easily expanded to fill the available time, however. 

Daniel Padwa is a technical manager at Goldman Sachs. This company uses a lot of computers: they have over 20,000 Windows desktops, 9000 Solaris servers, and around 2400 Linux servers. The bulk of the Linux machines are organized into compute farms running Monte Carlo simulations and such; they run a "hacked 2.4 kernel." There's a few hundred RHEL systems as well. Daniel expects the number of Linux systems to grow by an order of magnitude over the next few years - even though the company has no interest in desktop Linux systems. 

Most of the problems Goldman Sachs has with Linux happen above the kernel level; they include things like Java support, dealing with independent system vendors, configuration management, etc. They do, however, run into kernel-level problems as well. The biggest of these problems, by far, is storage. Goldman Sachs runs some 150 storage arrays adding up to about 1200 terabytes of space. Storage on this scale is expensive and a challenge to manage. Some of the issues that come up are: 

**Multipath support**. Numerous multipath implementations exist for Linux; some are free, and some proprietary. It would be nice, says Daniel, if there were one which "just works." In many cases, proprietary drivers are unavoidable. 

**Real-time, synchronous, remote mirroring** is a big deal. Financial institutions need serious disaster recovery plans in place. In practice, these plans include storage arrays which are mirrored, in real time, at a remote location. When a SCSI write to the local array completes, the system knows that it has also been committed to disk at the mirror site, many miles away. This mirroring makes up-to-the-second recovery possible, but it also adds a lot of latency to write operations. Maintaining performance then becomes a problem. 

**Driver interface stability**. When the driver API changes, proprietary drivers must be changed and, often, recertified. So the oft-heard request that internal APIs be made more stable, especially during a stable kernel series, was raised again. 

Driver interface issues are often best solved by putting the drivers into the mainline kernel. Free drivers tend to get fixed when an interface changes; proprietary drivers do not. Goldman Sachs understands that well, and has been pushing its vendors to release their drivers. In other words, big, commercial customers with no "free software" agenda are beginning to realize that their interests are best served by free drivers. _That_ is a crucial turning point which should help vendors understand that releasing their drivers will help their business. 

**Device name stability** is also an issue. Goldman Sachs runs with a great many disks installed; it is a real pain when a hardware change causes all of those disks to be renamed. 

A few issues beyond storage were raised; for example, **crash dumps** were requested as a "key" feature for tracking down problems. When you have thousands of systems, crashes will be a regular occurrence even if the systems are, individually, quite reliable. Crash dumps help in identifying patterns and recurring issues; these problems cannot be easily tracked down without this facility. The network dump scheme shipped by Red Hat is a step in the right direction, but it does not work well in a large network; what's really needed is a reliable dump mechanism which dumps to local disk. 

**Thread support**. The instability in the next generation pthread library has been a problem, especially for users running the "2.4 1/2" kernels shipped by certain distributors. In many areas, heavily threaded applications are the norm, and they need to be well supported. 

The Linux scheduler could still use some work as well. What some users really want is a process-fair scheduler which would group all threads in one task together and have them contend as a unit with the other tasks on the system. Scheduling each thread independently does not yield the results they need. The class-based kernel resource management work currently in progress may help with this. 

**NFS** , a traditional target of complaints for many years. The Linux NFS server is still not quite stable enough. The client works better, but still needs work, especially in its ability to recover from server failures. 

Finally, **long development periods** are a problem. It can take years for features added to a development kernel to find their way into supported distributions. There is also a lack of predictability in the kernel development process; nobody really knows when a particular kernel will stabilize. When particular features do stabilize, it would be nice to mark them as such (and then live up to it) so that vendors and users know that those features are now safe. 

Daniel also conveyed some confusion about how best to work with the kernel community. He is willing to dedicate some resources to fix SCSI layer problems, for example, but he is not looking to hire a SCSI developer for the long term. High-end users like Goldman Sachs have a hard time finding ways to help the community make the kernel better. 

The other customer at the panel was Amazon.com, represented by Willie Lewis. He stated that Amazon is predominantly a Linux operation; most parts of the organization had been switched over by last year. At least, on the operational side; he did not mention desktop deployments. Amazon runs a large number of big database clusters and NFS servers, all over the world. 

Willie also mentioned predictability problems with kernel development. There may well be a patch out there which solves one of his problems nicely, but he never really knows if that patch will be maintained into the future. Picking up new code, thus, is a bit of a leap of faith. 

Tracking down problems can be difficult. Crash dumps would help, as would some sort of (secure!) remote diagnostics mechanism. 

Willie did not mention it, but a developer sent by a large database company pointed out that asynchronous I/O was crucial for what Amazon is doing. Linus was unimpressed; he wants to actually _see_ AIO applications, and that means seeing the source. If Amazon's applications are so important, it is worth their while to push some of the code out into the world; it will help to ensure that the needed features will still work tomorrow. 

The customer panel was a useful addition to the summit. It is good to hear about the problems encountered by people trying to actually use Linux in interesting ways. Of course, this format is better suited to the problems of large companies than, say, little old ladies who are trying to organize their budgets. Even with that flaw, this sort of feedback path can only help to make the kernel better. 

[>> Next: Clustered storage](<https://lwn.net/Articles/94559/>).
