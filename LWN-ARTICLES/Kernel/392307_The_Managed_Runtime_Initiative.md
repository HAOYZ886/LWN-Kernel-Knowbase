---
title: The Managed Runtime Initiative
url: https://lwn.net/Articles/392307/
date: "June 16, 2010"
category: Managed Runtime Initiative
author: "By Jonathan Corbet June 16, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 16, 2010

The [Managed Runtime Initiative](<http://www.managedruntime.org/>) has recently [announced its existence](<http://www.azulsystems.com/about_us/press/azul-systems-launches-open-initiative-improving-managed-runtimes>). This group is dedicated to making "managed runtime" code (Java programs in particular) run faster on Linux systems. MRI's effort might not seem like a suitable topic for the Kernel Page, except for one thing: this group has just released thousands of lines of questionable code which, it claims, it plans to push upstream. 

The specific problem that the MRI people (actually Azul Systems employees) have set out to solve appears to be application pauses caused by garbage collection. Their solution is implemented at several levels, some of which are found in the kernel. For the curious, the patches can be found on [the MRI download page](<http://www.managedruntime.org/downloads>), helpfully packaged as a tarball filled with source-RPM files. They have also thoughtfully included all of Red Hat's patches; look for files containing "az" to pick the new stuff out of the noise. 

The first kernel patch adds an interface for loadable memory management modules. With this in place, loadable modules can create and claim their own VMAs which they manage. The Azul-supplied module creates a special device which provides a few dozen `ioctl()` operations for the management of memory within those VMAs. What is actually done by this module is on the obscure side; it involves dividing memory into "accounts" with names like "GC Pause Prevention." There appears to be code to provide transparent hugepage access to interested applications. There is also some sort of relaxed locking done within the special VMAs designed to improve scalability there. 

Then, there is the pluggable scheduler patch, creating a new `SCHED_ALT` scheduling class which sits between CFS and the realtime classes. The actual scheduler module's purpose is described as: 

The Azul scheduler is designed to provide a cpu resource guarantee on Linux: specifically that any process with 'committed' cpus and runnable threads available for those cpus will have its threads running on those cpus within 10ms. 

It allows the partitioning of the system into "committed" and ordinary CPUs, with special applications getting priority access to the committed CPUs. 

The MRI web page claims that ""it is the initiative's goal to upstream those related contributions into existing and complementary OSS projects (e.g. kernel.org and openjdk.org)"", but the kernel-related code has never, to your editor's knowledge, been seen on any kernel-related mailing list. It is heavy with `#ifdefs`, light on comments, and it adds exports for large numbers of low-level functions in the scheduler and VM code. Plus there is the little detail that the development community is unlikely to agree with this code's fundamental purpose. Pluggable schedulers have been rejected in the past; until now nobody has even dared to suggest pluggable memory management modules. 

In other words, we have a bunch of hackish code which was developed in total isolation; one wonders how many customers it has been shipped to. If Azul Systems and the MRI are serious about wanting to upstream it, they might just want to start talking with the development community fairly soon. One expects that they might just have a few changes to make.
