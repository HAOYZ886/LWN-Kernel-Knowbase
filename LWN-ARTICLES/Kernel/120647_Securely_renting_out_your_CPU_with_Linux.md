---
title: Securely renting out your CPU with Linux
url: https://lwn.net/Articles/120647/
date: "January 26, 2005"
category: Security
author: 
---

One might think that hacking kernels for SuSE would be enough to keep a person busy, but Andrea Arcangeli has more energy than that. So, on the side, he is working on a project called [cpushare](<http://www.cpushare.com/>). Essentially, it is a form of Linux-based worldwide grid network which would enable people to sell their idle CPU cycles to others who are in need of serious crunching power. Most systems sit idle most of the time; they might as well be working for a living instead. 

Before Linux users will accept outside code on their systems, they will need to be pretty well convinced that said code can be kept under control. Without some pretty fierce security, a grid network could quickly be turned into a serious zombie network, and that would be embarrassing for everybody involved. The long-term plan is to take advantage of "trusted computing" hardware to sandbox the guest code; that hardware will also be able to help prevent the owner of the system from interfering with (or even observing) the buyer's computations and data. For now, however, some other method of ensuring the security of the host systems will have to be devised. 

Andrea's answer is the [seccomp patch](<https://lwn.net/Articles/120192/>). It is, in fact, an exceedingly simple solution to the problem. With this patch in place, the kernel allows a process to make a one-way transition into a "secure computing" mode. A hook is placed in the system call gate which checks for this mode; when secure computing is turned on, an attempt to execute almost any system call will result in the immediate termination of the process. The only exceptions are `read()`, `write()`, `exit()`, and the two forms of `sigreturn()`. So the cpushare client would execute outside work by setting up some sockets to communicate with the master system, turning on the secure mode, and executing the client code. If all goes well, that code will be able to do nothing beyond pure CPU use and communicating through the sockets given to it. And exiting, of course. 

The objection that was raised is that this sort of sandboxing can be done from user space with `ptrace()`. At least, it can if [one little patch](<https://lwn.net/Articles/120680/>) is applied to ensure that the sandboxed process cannot continue if, somehow, the monitoring process dies first. Andrea [acknowledges](<https://lwn.net/Articles/120681/>) that the `ptrace()` solution is workable, but he still prefers the secure computing mode. The reason is simplicity: the seccomp patch is quite small, and it is relatively easy to verify that it does what is needed. The `ptrace()` code is rather more complex and harder to verify, even before considering the interactions with the user-space monitor. 

The public discussion was inconclusive, so it is hard to predict whether this patch will eventually be accepted or not.
