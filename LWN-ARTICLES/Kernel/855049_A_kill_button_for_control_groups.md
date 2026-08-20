---
title: "A \"kill\" button for control groups"
url: https://lwn.net/Articles/855049/
date: "May 3, 2021"
category: Control groups
author: "By Jonathan Corbet May 3, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 3, 2021

The kernel's [control-group mechanism](<https://lwn.net/Articles/603762/>) exists to partition processes and to provide resource guarantees (and limits) for each. Processes running within a properly configured control group are unable to deprive those running in a different group of their allocated resources (CPU time, memory, I/O bandwidth, etc.), and are equally protected from interference by others. With few exceptions, control groups are not used to take direct actions on processes; Christian Brauner's [cgroup.kill patch set](<https://lwn.net/ml/cgroups/20210503143922.3093755-1-brauner@kernel.org/>) is meant to be one of those exceptions. 

In current kernels, one way of acting on processes within a control group is through the "freezer", which can be used to suspend (or resume) all contained processes. Beyond that, though, there are few control-group knobs that will directly affect a process's state. Brauner's patch set adds another one in the form of a control file in each non-root group called `kill`; it ""does what it says on the tin"". Writing "1" to that file will cause the immediate death of every process contained within the group (more correctly, it causes the immediate delivery of a `SIGKILL` signal to each, which has a similar effect). If the control group contains other groups, those, too, will be exterminated. Once the operation is complete, the group will normally be left in an entirely depopulated state. 

There are a couple of exceptions to this behavior, of course. The kill operation is defined to work on a process; if the process contains many threads, they will all suffer the same fate. But, if the control group in question is operating in the [threaded mode](<https://lwn.net/Articles/729215/>), which allows the threads of a process to be split across multiple groups, that could lead to the untimely demise of threads that were not in the targeted group. So the kill operation will fail if attempted on groups running in the threaded mode. 

Similarly, the kill operation will not take down kernel threads, as that could lead to any of a number of surprising results. Writing to the `kill` file in a group containing kernel threads is allowed, but the kernel threads themselves will survive the operation. In such cases, the group will not be empty at the end. 

Brauner cites a number of potential uses for this feature. One of the most obvious ones is container management; if a decision is made that a container should go away, the kill operation is a quick and straightforward way to make that happen. Systemd organizes services into control groups already; it could use this operation as an easier way to stop a service when need be. Similarly, user-space out-of-memory managers could use it as a quick way to make entire control groups go away if the need arises. The kill operation could also be an effective fork-bomb defense; when the kill operation is invoked, a flag is set on the group that prevents the creation of new processes, stopping a forking process in its tracks. 

On the other hand, this feature could be thought of as equipping every control group in the system with a big red button with "do not push this" written on it. A stray write to the `kill` file has the potential to do a fair amount of damage to a running system. The obvious answer to such worries is "don't do that, then", but it is not hard to imagine some users wishing that there were some guard rails around this file. 

The current patch works by sending a `SIGKILL` signal to every process within the target group. There is not any provision for sending a different signal; that, too, seems like something that may be wanted at some point. The [`kill()`](<https://man7.org/linux/man-pages/man2/kill.2.html>) system call can send any signal; there may eventually be a case for allowing the `cgroup.kill` file to do the same. 

There have not been a lot of comments on this patch series so far, perhaps partly because it has not been circulated beyond the cgroups mailing list. There is probably not much to complain about with regard to the implementation, which is fairly straightforward, so if there is an issue that could slow this work down, it will have to do with the design of the feature itself. There seem to be clear use cases for it, though, so a kill switch may indeed be a control-group feature in the near future.
