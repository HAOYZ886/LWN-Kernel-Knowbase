---
title: Checkpoint/restart in user space
url: https://lwn.net/Articles/572125/
date: "October 29, 2013"
category: Checkpointing
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

There has long been a desire for the ability to checkpoint a set of processes (save their state to disk) and restore those processes at some future time, possibly on a different system. For almost as long, Linux has lacked that feature, but those days are coming to an end. Pavel Emelyanov led a session during the 2013 Kernel Summit's plenary day to update the audience on the status of this functionality. 

Pavel started with the history of this feature. [Early attempts](<https://lwn.net/Articles/375855/>) to add checkpoint/restart went with an entirely in-kernel approach. The resulting patch set was large and invasive; it looked like a maintenance burden and never got much acceptance from the broader development community. Eventually, some developers realized that the APIs provided by the kernel were nearly sufficient to allow the creation of a checkpoint/restore mechanism that ran almost [![\[Pavel Emelyanov\]](https://static.lwn.net/images/conf/2013/lce-ks/PavelEmelyanov-sm.jpg)](<https://lwn.net/Articles/572141/>) entirely in user space. All that was needed was a few additions here and there; as of the 3.11 kernel, all of those additions have been merged and user-space checkpoint/restart works. Live migration is supported as well. 

Pavel had some requests for developers designing kernel interfaces in the future. Whenever new resources are added to a process, he asked, please provide a call to query the current state. A classic example is timers; developers added interfaces to create and arm timers, but nothing to query them, so the checkpoint/restart developers had to fill that in. He also requested that any user-visible identifiers exposed by the kernel not be global; instead, they should be per-process identifiers like file descriptors. If identifiers must be global — he gave process IDs as an example — it will be necessary to create a namespace around them so that the same identifiers can be restored with a checkpointed process. 

Now that the basic functionality works, some interesting new features are being worked on. One of these checkpoints all processes in the system, but keeps the contents of their memory in place. It then boots into a new kernel with kexec and restores the processes quickly, using the saved memory whenever possible. This, Pavel said, is the path toward a seamless kernel upgrade. 

Andrew Morton expressed his amazement that all of this functionality works, especially given that the checkpoint/restore developers added very little in the way of new kernel code. Is there, he asked, anything that doesn't work? Pavel responded that they have tried a lot of stuff, including web servers, command-line utilities, huge high-performance computing applications, and more. Almost everything will checkpoint and restore just fine. 

Andrew then refined his question: could you write an application that is not checkpointable? The answer is "yes"; the usual problem is the use of external resources that cannot be checkpointed. For example, Unix-domain sockets where one end is held by a process that is not being checkpointed will block things; syslog can apparently be a problem in this regard. Work is being done to solve this problem for a set of known services; the systemd folks want it, Pavel added. Unknown devices are another problematic resource; there is a library hook that can be used to add support for specific devices if their state can be obtained and restored. 

Beyond that, though, this long-sought functionality seems to work at last. 

[Next: [A kernel.org update](<https://lwn.net/Articles/572127/>)].
