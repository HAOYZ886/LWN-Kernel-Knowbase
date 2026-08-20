---
title: 4K Stacks in 2.6
url: https://lwn.net/Articles/84583/
date: "May 12, 2004"
category: Kernel stack
author: 
---

Traditionally, the Linux kernel has used 8KB kernel stacks on most architectures. That stack must suffice for any sequence of calls that may result from a system call - plus the needs of any (hard or soft) interrupt handlers that may be invoked at the same time. In practice, stack overflows are pretty much unheard of in stable kernels; the kernel developers have long since learned to avoid large automatic variables, recursive functions, and other things which can use large amounts of stack space. 

There have been patches circulating for some time now which reduce the kernel stack to 4KB. It is generally understood that the switch to smaller stacks will happen at some point; as a result, much work has recently gone into finding code paths in the kernel which are overly stack-hungry. Part of that effort is simply lots of testing; for that reason, recent -mm kernels no longer even offer an 8KB stack option. The hope is that, if enough people try out the smaller stacks and shake out the bugs, 4KB stacks can be merged into 2.6 in the near future. 

The smaller stacks are scary to some people; it is hard to be certain that all of the possible paths through the kernel have actually been tested. 4KB stacks also break binary modules, and the nVidia drivers in particular. So there is a certain amount of pressure to defer this change into 2.7. 

One might well wonder why the kernel hackers are trying to put this sort of change into a stable kernel series. The problem with 8KB stacks is that they require an "order 1" memory allocation: two pages which are contiguous in physical memory. Order 1 allocations can be very hard to satisfy once the system has been running for a while; physical memory can become so fragmented that two adjacent free pages simply do not exist. The kernel will try hard to free up pages to satisfy larger allocations; the result can be a slow, painful, thrashing system. 

Each process on the system has its own kernel stack, which is used whenever the system goes into kernel mode while that process is running. Since each process requires a kernel stack, the creation of a new process requires an order 1 allocation. So the two-page kernel stacks can limit the creation of new processes, even though the system as a whole is not particularly short of resources. Shrinking kernel stacks to a single page eliminates this problem and makes it easy for Linux systems to handle far more processes at any given time. 

Arjan van de Ven also [made the interesting claim](<https://lwn.net/Articles/84592/>) that the 4KB stacks are actually safer. His reasoning has to do with one other aspect of the 4KB stack patch: it moves interrupt handling onto a separate, dedicated stack. Software interrupts also get their own stack. Since interrupt handling has been moved away from the per-process kernel stack, the amount of space for system call handling remains about the same, and the stack space for interrupts has been increased. 

The final decision on the integration of 4KB stacks has not yet been made; there are, seemingly, a few problems which still need to be tracked down. If things settle out, however, this fairly significant change could yet be merged into 2.6.
