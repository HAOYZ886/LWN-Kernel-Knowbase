---
title: "Uprobes: not quite there yet"
url: https://lwn.net/Articles/370322/
date: "January 20, 2010"
category: Tracing; Uprobes; Utrace
author: "By Jonathan Corbet January 20, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 20, 2010

Tracing support in Linux has made a great deal of progress over the course of the last year or so. One important feature still lacks support in the mainline kernel, though: seamless, combined tracing of user-space execution along with the kernel. The subsystem which is meant to support this feature - utrace - has [run into a number of roadblocks](<http://lwn.net/Articles/325180/>) on its way into the mainline. Now a higher-level feature, [uprobes](<http://lwn.net/Articles/369358/>), has been proposed as a solution for dynamic probing of user-space programs. All told, the combination shows a lot of progress toward inclusion, but the resulting discussion suggests that there are still problems to be overcome before this code will be merged. 

This version of uprobes is actually two independent modules which address the problem at different levels. The lower-level piece is called "UBP," for user-space break points; its job is to handle the actual placement of probes into user-space processes. The developers reasoned that there might be additional users of user-space probes in the future, so the facilities for the placement and removal of those probes were carved out separately. 

On top of UBP is the actual uprobes code, which handles higher-level details. Uprobes arbitrates between multiple users of breakpoints, even if two users want to place a probe at the same location. It uses utrace to ensure that processes are not running in an area which is about to have a probe inserted, and deals with the case of multiple processes running the same code where some are being traced and others are not. The uprobe code is also in charge of actually calling the probe function when a probe is hit and recovering properly if that function behaves poorly. 

This separation is the first point of contention; Peter Zijlstra (who has been the main reviewer of this code so far) sees uprobes as an unnecessary glue layer which could be eliminated. Peter would rather see any needed features pushed down into UBP, after which the higher-level code could be dropped. The uprobes developers disagree, though, saying that the functions implemented at that level are necessary and cannot really be eliminated. This part of the discussion kind of died out, but it doesn't look like the developers are inclined to make major changes here. 

The next problem is with the implementation of the probes themselves. When a probe is placed in a user-space program, the instruction at the probed location is overwritten by a breakpoint. When the breakpoint is hit, the probe handler function is invoked; once it returns, the replaced instruction must be executed somehow. A simple implementation would put that instruction back into its original location, single-step through it, then restore the breakpoint once again. That approach fails, though, if there is a second process (or thread) running the probed code. If that second process executes through the probed area while the probe has been removed, the associated event will be lost. 

So the uprobes developers took a separate approach, called "single-step out of line" or "execute out of line" (XOL). A separate region of memory is set up for the purpose of holding instructions which have been displaced by probe breakpoints. When one of those instructions is to be executed, it is run (again, in single-step mode) out of this separate area; after that, control returns after the probe location. This solution allows a probe to work with multiple processes at the same time. 

The problem is this: the memory containing the XOL instructions must be in the probed process's address space. So the XOL code adds a virtual memory area (VMA) to the process, reserving a range of address space for this purpose. This works, but it strikes some observers as inelegant at best, and potentially disruptive at worst. Currently, the layout of a process's address space is almost entirely under the control of the process itself. The injection of a special kernel VMA can perturb the process's control of its address space, causing other VMAs to move or conflicting with an attempt by the process to place a VMA at a specific location. Debuggers are often known to distort application behavior (leading to "heisenbugs" which disappear when somebody attempts to observe them directly), but tracing, which is meant to work on production systems, should really minimize such distortions. Peter also dislikes the precedent of kernel code messing with a process's address space. Finally, on 32-bit systems, losing even a small amount of address space to a kernel function is likely to be unwelcome in a number of situations. 

Solving this problem is not necessarily easy. Peter seems to favor emulating the displaced instruction, but that would require the implementation of a full instruction emulator in the kernel. That code would be large, architecture-specific, and error prone. There was some discussion of trying to run the instruction in kernel space, but doing that securely appears to be a challenging task. After an extended discussion, the prevailing opinion seemed to be something like that [expressed](<https://lwn.net/Articles/370323/>) by Pekka Enberg: 

I guess we're looking at few megabytes of the address space for normal scenarios which doesn't seem too excessive... I don't like the idea but if the performance benefits are real (are they?), maybe it's a worthwhile trade-off. 

In the end, perhaps the kernel developers will hold their noses and merge this approach, but chances are they'll need to talk about it for a while yet first. 

The uprobes code comes with an ftrace plugin which provides an interface to user space for the placement and management of probes. The problem here is that the kernel developers have, for all practical purposes, decided that there will be no more ftrace plugins added to the kernel. New features are supposed to go through the perf events subsystem instead, which is seen as having a better-designed interface. So the current ftrace plugin will almost certainly have to be redone for perf events before this code can go in. 

The ftrace plugin also associates user-space probes with specific process of interest. Peter argues that it makes more sense to hook probes onto executable files, then make the process association by way of the VMA structure when the file is mapped. Existing features in the kernel, perhaps supplemented with a simple hook or two, would make it easy for uprobes to find processes running code from a file and to deal with process comings and goings while the probes are in place. The uprobes developers have not said as much, as of this writing, but it seems likely that the API could be reworked in those terms. 

Then, there is the nagging issue of the utrace layer, which has not yet found its way into the mainline. It has recently been added to linux-next, but there is [some discomfort](<https://lwn.net/Articles/370521/>) with that and it's not clear if it will remain there or not. 

All of this may seem like a lot of obstacles to the merging of this code, but it also represents a step forward. The road into the mainline has been long for utrace; a final detour or two seems about par for the course. The existence of uprobes as an in-kernel user of utrace might help its cause, once uprobes itself passes muster. Assuming consensus on these issues can be reached, it should be possible to make a last round of changes and be quite close to getting the code merged \- though it might be difficult to get this done for the 2.6.34 merge window. But, if things go well, we should have user-space probing not too much later than that.
