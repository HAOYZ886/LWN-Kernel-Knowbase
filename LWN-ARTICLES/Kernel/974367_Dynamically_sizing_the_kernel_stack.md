---
title: Dynamically sizing the kernel stack
url: https://lwn.net/Articles/974367/
date: "May 21, 2024"
category: Kernel stack
author: "By Jonathan Corbet May 21, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 21, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The kernel stack is a scarce and tightly constrained resource; kernel developers often have to go far out of their way to avoid using too much stack space. The size of the stack is also fixed, leading to situations where it is too small for some code paths, while wastefully large for others. At the [2024 Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Pasha Tatashin proposed making the kernel stack size dynamic, making more space available when needed while saving memory overall. This change is not as easy to implement as it might seem, though. 

Every thread has its own kernel stack, he began. The size of each thread's kernel stack was 8KB for a long time, and it is still that way on some architectures, but it was [increased to 16KB](<https://lwn.net/Articles/600644/>) on x86 systems in 2014. That change, which was driven by stack overflows experienced with subsystems like KVM and FUSE, makes every thread in the system heavier. 

Expanding the stack size to 16KB has not solved all of the problems, though; kernel code is using more stack space in many contexts, he said. I/O is becoming more complex, perf events are handled on the stack, compilers are more aggressively inlining code, and so on. Google has stuck with 8KB stacks internally for memory efficiency, but is finding that to be increasingly untenable, and will be moving to 16KB stacks. That, he said, is an expensive change, causing an increase in memory use measured in petabytes. There are applications that create millions of threads, each of which pays the cost for the larger kernel stack, but 99% of those threads never use more than 8KB of that space. 

[![\[Pasha Tatashin\]](https://static.lwn.net/images/conf/2024/lsfmm/PashaTatashin-sm.png)](<https://lwn.net/Articles/974374/>) Thus, he proposed making the kernel stack size dynamic; each thread would start with a 4KB stack, which would be increased in response to a page fault should that space be exhausted. An initial implementation was [posted in March](<https://lwn.net/ml/linux-kernel/20240311164638.2015063-1-pasha.tatashin@soleen.com/>). The proposed solution takes advantage of [virtually mapped stacks](<https://lwn.net/Articles/692953/>), which make it relatively easy to catch overflows. A larger stack is allocated in the kernel page tables, but only one 4KB page is mapped. The result is a significant speedup because the kernel does not have to find as much memory for kernel stacks, and tests have shown a 70-75% savings in memory used for the stacks. That, he said, was from a "simple boot test"; tests with real workloads would have shown a larger savings. 

There is an interesting challenge associated with page faults for stack access, though: page faults are also handled on the kernel stack, which has just run out of space. When a thread tries to access an unmapped page and causes a page fault, the fault handler will try to save the current processor state onto the kernel stack, which will cause a double fault. The x86 architecture does not allow handling double faults; code is simply supposed to abort and clean up when that happens. If the kernel tries, instead, to handle that fault and expand the stack, it is operating outside of the rules defined by the architecture, and that tends not to lead to good things. 

Solutions to that problem seem to be expensive. One idea, suggested by Matthew Wilcox but also already present on Tatashin's slides, is to add an `expand_stack()` function that would be called by subsystems that know they will need more stack space. It would map the extra space ahead of its use, avoiding the double-fault situation. Michal Hocko responded that this solution seemed like a game of [Whac-A-Mole](<https://en.wikipedia.org/wiki/Whac-A-Mole>), with developers trying to guess where the stack might overflow. But direct reclaim, which can call filesystem or I/O-related functions with deep stack use, can happen just about anywhere. If that causes an overflow, the system will panic. 

A second possible solution, Tatashin said, would be to take advantage of some of the kernel-hardening work to automatically grow the stack as needed. Specifically, he would like to use the [STACKLEAK](<https://lwn.net/Articles/764325/>) mechanism, which uses a GCC plugin to inject stack-size checks into kernel functions as they are compiled. That code could be enhanced to automatically grow the stack when usage passes a threshold. This solution adds almost no overhead to systems where STACKLEAK is already in use — but it is rather more expensive if STACKLEAK is not already enabled. 

Finally, a third option would be to limit dynamic stacks to systems that either do not save state to the stack on faults or that can handle double faults. Tatashin suggested that x86 systems with [FRED](<https://www.intel.com/content/www/us/en/content-details/779982/flexible-return-and-event-delivery-fred-specification.html>) support might qualify, and 64-bit Arm systems as well. 

Time for the session was running short as Hocko said that he liked the second solution but wondered what the cost would actually be. Tatashin said that he has been working on reducing it; he has refactored the STACKLEAK code to be more generic, so that it can be used for this purpose without needing to include the hardening features. A stack-frame size can be set at build time, and the plugin will only insert checks for functions that exceed that size. David Hildenbrand said that this scheme could be thwarted by a long chain of calls to small functions; Hocko said that would make the feature less than fully reliable. Tatashin answered that there is probably at least one large function somewhere in any significant call chain, but Hocko said that is not necessarily the case with direct reclaim. 

Steve Rostedt said that, perhaps, the frame-size parameter could be set to zero, causing the check be made at every function call; Tatashin answered that, while he has not measured the overhead of the check, it would certainly add up and be noticeable in that case. The final suggestion of the session came from Hocko, who said that perhaps the ftrace hooks could be used instead of the STACKLEAK infrastructure, but Rostedt said that option would be too expensive to be worth considering.
