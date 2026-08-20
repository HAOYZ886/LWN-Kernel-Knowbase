---
title: "KS2007: Containers"
url: https://lwn.net/Articles/249080/
date: "September 10, 2007"
category: Containers
author: "By Jonathan Corbet September 10, 2007 LWN.net Kernel Summit 2007 coverage"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 10, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

For the first time in a few years, virtualization was not on the agenda at the 2007 kernel summit. The related field of containers, however, was deemed worth talking about. The virtualization problem has been mostly solved, at least at the kernel level, but there is still a lot of work to do in the containers area. 

Paul Menage talked about the [process containers patch](<http://lwn.net/Articles/236038/>), which has recently been rebranded "control groups." The control groups API is currently being used by the CFS scheduler, cpusets, and the memory controller code. Work in progress includes rlimits and an interface to the process freezer used by the suspend/resume code. Controlling the freezer via control groups allows user space to freeze specific groups of processes, which, in turn, is very useful when implementing checkpointing and live migration. In particular, with control groups, it will be possible to freeze an entire group of processes in an atomic way. 

Control groups have very little overhead when not in use. There is an approximately 1% hit on the `fork()` and `exec()` calls when control groups are being used. The control groups code is managed by way of a virtual filesystem. This filesystem is a user-space API which must be managed carefully; there needs to be consistency across the various controllers which can work with control groups. To that end, parts of this interface are being pushed into generic code when possible. One other issue is the use of control groups within containers. It would be nice if a containerized system could manage control groups for processes within the container, but that is not yet implemented. 

Eric Biederman talked about the container situation in general. Implementing containers requires the creation of container-specific namespaces for all of the global resources found on the system. Namespaces for time, SYSV interprocess communication primitives, and users are in the mainline now. There is a process ID namespace patch in -mm which is getting close. Network namespaces are in development now. Resources which still need to have namespaces created for them include system time (important to keep time from moving backward when containers are migrated from one system to another) and devices. 

Each namespace which is created requires an option to the `clone()` system call to say whether it should be shared or not. It seems that there may not be enough clone bits to go around; how that problem will be solved is not clear. 

So, how close are we to having a working container solution? It is still somewhat distant, says Eric. But, when it's done, the support for containers in Linux will be more general and more capable than the options which are available now. It is, he says, a more general solution than OpenVZ, and, unlike Solaris Zones, it will have network namespaces. An important milestone will be the incorporation of PID namespaces, which will make it possible to start actually playing with Linux containers. That code should, with luck, be merged before too long, though it is proving to be a bit of a challenge: kernel code has process IDs hidden away in a number of unexpected places. 

Stay tuned; perhaps, by the next kernel summit, containers will be considered to be a solved problem as well.
