---
title: "Kernel Summit 2005: Realtime capabilities"
url: https://lwn.net/Articles/144278/
date: "July 20, 2005"
category: Realtime
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
Paul McKenney and Ingo Molnar ran the session on the implementation of realtime capabilities in Linux. Given the discussions which have taken place on the mailing lists over the last year, one might have expected this session to be one of the more contentious ones, but that turned out not to be the case. Instead, the available time was mostly taken up with a summary of the options available now and how they might be judged. 

There are, says Paul, several features which one might look for in a realtime implementation. They are: 

  * Quality of service. What types of services are supported, what sort of deadlines are promised, and what is the probability of meeting those deadlines? Also: what are the performance impacts of the solution? 

  * The amount of global knowledge required. How much of the system must be audited to be sure that the system will behave as required? 

  * The API provided. Can the system be programmed with Unix-like calls, or is something else required? 

  * Complexity: how much is contained within the system, and how much is pushed into the application? 

  * Fault isolation. If something goes wrong somewhere in the system, what are the chances that everything else will continue running correctly? 

  * Supported configurations. Which hardware and software can be used? 

Several approaches were listed: the vanilla Linux kernel, the kernel preemption option, a nested operating system (like RTLinux), dual operating system solutions, the realtime preemption patch, migration between operating systems (RTAI-Fusion, for example), or migration within the operating system (solutions which reserve a CPU for realtime tasks). Paul's chart was dense and hard to read from the back of the room; your editor will not attempt to reproduce it here. In any case, there was no real discussion of the merits of the various solutions. 

There are other things which might need to be looked at to provide a full realtime implementation. High-resolution timers, for example. The "variable sleep time" and dynamic tick patches (see [this Kernel Page article](<http://lwn.net/Articles/138969/>)) are another, as is the [FUSYN](<http://lwn.net/Articles/67194/>) patch (which implements user-space mutexes with priority inheritance). Looking further ahead, realtime systems might need features like deterministic I/O and wider use of priority inheritance - in memory allocations, for example. Again, there was no time to actually discuss these thoughts.
