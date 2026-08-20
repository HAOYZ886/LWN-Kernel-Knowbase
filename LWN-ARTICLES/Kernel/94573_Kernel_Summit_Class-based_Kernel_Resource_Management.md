---
title: "Kernel Summit: Class-based Kernel Resource Management"
url: https://lwn.net/Articles/94573/
date: "July 21, 2004"
category: "Class-based resource management; Resource management"
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
The [class-based kernel resource management](<http://ckrm.sourceforge.net/>) work was presented by Rik van Riel and Shailabh Nagar. The basic idea behind CKRM is that it: 

  1. Organizes processes and sockets into groups. This organization can be done in a number of ways using pluggable classifier modules. Typical schemes include the user on whose behalf things are running or the program that is being run. 

  2. Applies a policy on how much of the system each class is allowed to use. Resource usage which can be controlled includes CPU usage, memory usage, I/O bandwidth, etc. 

The whole thing is configured through a virtual filesystem; creating a new group is just a matter of making a new directory in that filesystem. The normal Unix permissions apply in this directory; depending on how they are set, non-root users can make changes to some or all resource policies. 

Uses for this mechanism include workload consolidation (restricting parts of the system's workloads to a given amount of resource usage), quality of service guarantees for network services or individual users, etc. CKRM can limit the amount of memory used by OpenOffice (_something_ has to do that) or give a database manager process priority access to the machine. 

Linus objected to the term "guarantees," claiming that any attempt to provide resource guarantees will lead to poor performance, deadlocks, or both. 

Alternatives to CKRM were quickly presented. Virtualization works for some sorts of resource limitation tasks, but do not work well on the desktop and can suffer from latency problems. Various user-mode solutions, such as "zapper daemons," are unable to respond to quick surges in resource use. 

There were various objections to the CKRM implementation; some called it over-engineered. Linus would like to see the general resource classes split into separate classes for every type of resource which is being controlled. It may be desirable to put a process into one class for its CPU usage, but another one altogether for controlling its I/O bandwidth. Various other implementation changes were requested as well. CKRM will likely find its way into the kernel at some point, but it will likely need another iteration or two through the developer review process first. 

[>> Next: OSDL relations](<https://lwn.net/Articles/94574/>).
