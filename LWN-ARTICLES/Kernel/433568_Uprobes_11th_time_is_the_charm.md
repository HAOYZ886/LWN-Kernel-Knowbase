---
title: "Uprobes: 11th time is the charm?"
url: https://lwn.net/Articles/433568/
date: "March 16, 2011"
category: Tracing; Uprobes
author: "By Jonathan Corbet March 16, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 16, 2011

Last week's Kernel Page included [an article about improving the `ptrace()` interface](<https://lwn.net/Articles/432114/>); the author of that work, Tejun Heo, was quoted as saying that part of the problem with `ptrace()` is that it has been starved of developer attention in favor of efforts to replace it entirely. One of those efforts is uprobes, which has also been featured on this page a few times. [A new uprobes patch](<https://lwn.net/Articles/433327/>) was posted on March 14; so this seems like a good time to have a look at it and further deprive `ptrace()` of attention. Uprobes looks like it is getting closer to acceptance, but it seems unlikely that the 11th revision will be the last. 

The purpose of the uprobes subsystem is what one might expect: to enable the placement of probes into user-space executable process memory. These probes might be used to support a debugger like gdb (though uprobes is [said](<https://lwn.net/Articles/433572/>) to be unsuitable for use by gdb in its current form) or to support user-space tracing. This feature does thus duplicate some of the functionality provided by `ptrace()`, which will make its acceptance harder, especially since `ptrace()` is (more or less) a standardized interface. To succeed, uprobes will clearly have to do things better than `ptrace()` does. 

The `ptrace()` interface is tied to processes; uprobes, instead, works with files. A probe is placed at a certain offset within a specific file; it will then trigger for every process which executes through the probe's location. If the code placing the probe is only interested in specific processes, it will need to filter the events itself. The interface may seem a little strange - users will probably almost always be interested in specific processes - but there are some advantages to doing things this way. 

Underneath the hood, uprobes works by faulting in the page which will contain the probe. The instruction at the probe location is copied aside and replaced by a breakpoint. Every process which has that file mapped then gets a pointer in its `mm` structure pointing to the data describing the probe(s) for that file. When a process executes the breakpoint, the probe's handler function will be called; on that handler's return, the kernel will single-step the displaced instruction, then return to the location following the probe. 

This "execute out of line" (XOL) mechanism has been controversial in the past because it requires the injection of a new virtual memory area (VMA) into every process which encounters probes. That VMA is seen as a distortion of the process's behavior which could have strange effects. The alternatives, though, are not entirely appealing either. The `ptrace()` approach is to put the original instruction back into its original location, execute it, then replace the breakpoint; that only works if every process which has the file mapped is stopped for the duration of the operation (otherwise they might execute the affected code while the breakpoint is missing). Uprobes, instead, is able to handle breakpoint hits without perturbing other processes. Another alternative discussed in the past is emulating the displaced instruction in the kernel; that requires having a full x86 emulator in kernel space, which is not entirely appealing either. So the current plan appears to be to stick with XOL. 

Not having to stop the world when a breakpoint is hit is one of the advantages of uprobes, but there are others. It dispenses with the whole `ptrace()` mechanism involving signals, reparenting processes, and so on. Handling a probe hit does not require a context switch unless the probe itself does; many types of tracing tasks, for example, would never have to switch to another process. Uprobes also allows multiple applications to be tracing the same set of processes at the same time. All of these make the interface appealing to some users. 

Who those users are is not clear to everybody, though. There is clearly some interest in the SystemTap camp, but the needs of SystemTap do not necessarily carry a lot of weight on linux-kernel. Thomas Gleixner [put it this way](<https://lwn.net/Articles/433582/>): 

And it does not matter at all whether systemtap can use this or not. If the main debuggers used like gdb are not going to use it then it's a complete waste. We don't need another debugging interface just for a single esoteric use case. 

At times, gdb developers have [indicated](<https://lwn.net/Articles/433895/>) that they might be open to using a Linux-specific interface if there were advantages to doing so. Such use seems distant at the moment, though. More immediate users are likely to be found in the tracing community; uprobes opens the possibility of getting single stream of trace data covering both user and kernel space. `ptrace()` is not a useful interface for tracing, so something needs to be done (though there is still some disagreement over whether user-space tracing needs to involve the kernel at all). Uprobes might be that something. 

In fact, this version of the uprobes patch includes an ftrace-based interface. Part 20 of the patch contains the entirety of the documentation for this feature, quoted below: 

```
# cd /sys/kernel/debug/tracing/
        # cat /proc/`pgrep  zsh`/maps | grep /bin/zsh | grep r-xp
        00400000-0048a000 r-xp 00000000 08:03 130904 /bin/zsh
        # objdump -T /bin/zsh | grep -w zfree
        0000000000446420 g    DF .text  0000000000000012  Base        zfree
        # echo 'p /bin/zsh:0x46420 %ip %ax' > uprobe_events
        # cat uprobe_events
        p:uprobes/p_zsh_0x46420 /bin/zsh:0x0000000000046420
        # echo 1 > events/uprobes/enable
        # sleep 20
        # echo 0 > events/uprobes/enable
        # cat trace
```

An actual document is listed as a "TODO" item. The current interface looks a bit painful to use, and it appears to be limited to printing register contents for now. A more flexible and better documented interface could prove useful, though, especially if (as planned) it also can be made to work with the perf events subsystem. 

The comments on the patch set indicate some concern about whether the kernel needs the feature or not. But even the more critical reviewers have been going over the code pointing out small things - the kind of review one does when one wants to help the author get the code into shape for merging. This code will not be merged for 2.6.39, and, for this type of code, making predictions for merging at any definite time is a hazardous affair. But, given sufficient will, it seems like uprobes could be made ready for inclusion sometime this year.
