---
title: Seccomp and sandboxing
url: https://lwn.net/Articles/332974/
date: "May 13, 2009"
category: "Ftrace; Security-seccomp"
author: "By Jonathan Corbet May 13, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 13, 2009

Back in 2005, Andrea Arcangeli, mostly known for memory management work in those days, [wandered into the security field](<http://lwn.net/Articles/120647/>) with the "secure computing" (or "seccomp") feature. Seccomp was meant to support [a side business of his](<http://cpushare.com/>) which would enable owners of Linux systems to rent out their CPUs to people doing serious processing work. Allowing strangers to run arbitrary code is something that people tend to be nervous about; they require some pretty strong assurance that this code will not have general access to their systems. 

Seccomp solves this problem by putting a strict sandbox around processes running code from others. A process running in seccomp mode is severely limited in what it can do; there are only four system calls - `read()`, `write()`, `exit()`, and `sigreturn()` \- available to it. Attempts to call any other system call result in immediate termination of the process. The idea is that a control process could obtain the code to be run and load it into memory. After setting up its file descriptors appropriately, this process would call: 

```
prctl(PR_SET_SECCOMP, 1);
```

to enable seccomp mode. Once straitjacketed in this way, it would jump into the guest code, knowing that no real harm could be done. The guest code can run in the CPU and communicate over the file descriptors given to it, but it has no other access to the system. 

Andrea's CPUShare never quite took off, but seccomp remained in the kernel. Last February, when a security hole was found in the seccomp code, Linus [wondered](<https://lwn.net/Articles/332976/>) whether it was being used at all. It seems likely that there were, in fact, no users at that time, but there was one significant prospective user: Google. 

Google is not looking to use seccomp to create a distributed computing network; one assumes that, by now, they have developed other solutions to that problem. Instead, Google is looking for secure ways to run plugins in its [Chrome](<http://www.google.com/chrome>) browser. The Chrome [sandbox](<http://dev.chromium.org/developers/design-documents/sandbox>) is described this way: 

Sandbox leverages the OS-provided security to allow code execution that cannot make persistent changes to the computer or access information that is confidential. The architecture and exact assurances that the sandbox provides are dependent on the operating system. Currently the only finished implementation is for Windows. 

It seems that the Google developers thought that seccomp would make a good platform on which to create a "finished implementation" for Linux. Google developer Markus Gutschke [said](<https://lwn.net/Articles/332982/>): 

Simplicity is really the beauty of seccomp. It is very easy to verify that it does the right thing from a security point of view, because any attempt to call unsafe system calls results in the kernel terminating the program. This is much preferable over most ptrace solutions which is more difficult to audit for correctness. 

The downside is that the sandbox'd code needs to delegate execution of most of its system calls to a monitor process. This is slow and rather awkward. Although due to the magic of clone(), (almost) all system calls can in fact be serialized, sent to the monitor process, have their arguments safely inspected, and then executed on behalf of the sandbox'd process. Details are tedious but we believe they are solvable with current kernel APIs. 

There is, however, the little problem that sandboxed code can usefully (and safely) invoke more than the four allowed system calls. That limitation can be worked around ("tedious details"), but performance suffers. What the Chrome developers would like is a more flexible way of specifying which system calls can be run directly by code inside the sandbox. 

One suggestion that came out was to add a new "mode" to seccomp. The API was designed with the idea that different applications might have different security requirements; it includes a "mode" value which specifies the restrictions that should be put in place. Only the original mode has ever been implemented, but others can certainly be added. Creating a new mode which allowed the initiating process to specify which system calls would be allowed would make the facility more useful for situations like the Chrome sandbox. 

Adam Langley (also of Google) has posted [a patch](<http://lwn.net/Articles/332438/>) which does just that. The new "mode 2" implementation accepts a bitmask describing which system calls are accessible. If one of those is `prctl()`, then the sandboxed code can further restrict its own system calls (but it cannot restore access to system calls which have been denied). All told, it looks like a reasonable solution which could make life easier for sandbox developers. 

That said, this code may never be merged because the discussion has since moved on to other possibilities. Ingo Molnar, who has been arguing for the use of the ftrace framework in a number of situations, [thinks that ftrace is a perfect fit](<https://lwn.net/Articles/332986/>) for the Chrome sandbox problem as well. He might be right, but only for a version of ftrace which is not, yet, generally available. 

Using ftrace for sandboxing may seem a little strange; a tracing framework is supposed to report on what is happening while perturbing the situation as little as possible. But ftrace has a couple of tools which may be useful in this situation. The system call tracer is there now, making it easy to hook into every system call made by a given process. In addition, the current development tree (perhaps destined for 2.6.31) includes an event filter mechanism which can be used to filter out events based on an arbitrary boolean expression. By using ftrace's event filters, the sandbox could go beyond just restricting system calls; it could also place limits on the arguments to those system calls. An [example](<https://lwn.net/Articles/332987/>) supplied by Ingo looks like this: 

```
{ "sys_read",		"fd == 0" },
        { "sys_write",		"fd == 1" },
        { "sys_sigreturn",		"1" },
        { "sys_gettimeofday",	"tz == NULL" },
```

These expressions implement something similar to mode 1 seccomp. But, additionally, `read()` is limited to the standard input and `write()` to the standard output. The sandboxed process is also allowed to call `gettimeofday()`, but it is not given access to the time zone information. 

The expressions can be arbitrarily complex. They are also claimed to be very fast; Ingo claims that they are quicker than the evaluation of security module hooks. And, if straight system call filtering is not enough, arbitrary tracepoints can be placed elsewhere. All told, it does seem like a fairly general mechanism for restricting what a given process can do. 

The problem cannot really be seen as solved yet, though. The event tracing code is very new and mostly unused so far. It is out of the mainline still, meaning that it could easily be a year or so until it shows up in kernels shipped by distributions. The code allowing this mechanism to be used to control execution is yet to be written. So Chrome will not have a sandbox based on anything other than mode 1 seccomp for some time (though the Chrome developers are also evaluating using SELinux for this purpose). 

Beyond that, there are some real doubts about whether system call interception is the right way to sandbox a process. There are well-known difficulties with trying to verify parameters if they are stored in user space; a hostile process can attempt to change them between the execution of security checks and the actual use of the data. There are also interesting interactions between system calls and multiple ways to do a number of things, all of which can lead to a leaky sandbox. All of this has led James Morris to [complain](<https://lwn.net/Articles/332990/>): 

I'm concerned that we're seeing yet another security scheme being designed on the fly, without a well-formed threat model, and without taking into account lessons learned from the seemingly endless parade of similar, failed schemes. 

Ingo is not worried, though; he notes that the ability to place arbitrary tracepoints allows filtering at any spot, not just at system call entry. So the problems associated with system call interception are not necessarily an issue with the ftrace-based scheme. Beyond that, this is a specific sort of security problem: 

Your argument really pertains to full-system security solutions - while maximising compatibility and capability and minimizing user inconvenience. _That_ is an extremely hard problem with many pitfalls and snake-oil merchants flooding the roads. But that is not our goal here: the goal is to restrict execution in very brutal but still performant ways. 

This has the look of a discussion which will take some time to play out. There is sure to be opposition to turning the event filtering code into another in-kernel security policy language. It may turn out that the simple seccomp extension is more generally palatable. Or something completely different could come along. What is clear is that the sandboxing problem is hard; many smart people have tried to implement it in a number of different ways with varying levels of success. There is no assurance that that the solution will be easier this time around.
