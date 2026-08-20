---
title: Reorganizing the address space
url: https://lwn.net/Articles/91829/
date: "June 30, 2004"
category: "Memory management-User-space layout"
author: 
---

![\[memory layout diagram\]](https://static.lwn.net/images/ns/kernel/mmap1.png) The traditional organization of the virtual address space (as seen from user space, on x86 systems) is as shown in the diagram to the right. The very bottom part of the address space is unused; it is there to catch NULL pointers and such. Starting at 0x8000000 is the program text - the read-only, executable code. The text is followed by the heap region, being the memory obtainable via the `brk()` system call. Typically functions like `malloc()` obtain their memory from this area; non-automatic program data is also stored there. 

The heap differs from the first two regions in that it grows in response to program needs. A program like `cat` will not make a lot of demands on the heap (one hopes), while running a `yum update` can grow the heap in a truly disturbing way. The heap can expand up to 1GB (0x40000000), at which point it runs into the mmap area; this is where shared libraries and other regions created by the `mmap()` system call live. The mmap area, too, grows upward to accommodate new mappings. 

Meanwhile, the kernel owns the last 1GB of address space, up at 0xc0000000. The kernel is inaccessible to user space, but it occupies that portion of the address space regardless. Immediately below the kernel is the stack region, where things like automatic variables live. The stack grows downward. On a really bad day, the stack and the mmap area can run into each other, at which point things start to fail. 

This organization has worked for some time, but it does have a couple of disadvantages. It fragments the address space, such that neither the heap nor the mmap area can make use of the entire space. If one program makes heavy use of the heap, it could run out of memory, even though a large chunk of space is available between the mmap area and the stack. Normally, not even `yum` can occupy that much heap, but there are other applications out there which are up to that challenge. 

![\[revised memory layout\]](https://static.lwn.net/images/ns/kernel/mmap2.png) As a way of making life safer for the true memory hogs out there, Ingo Molnar has posted [a patch](<https://lwn.net/Articles/90311/>) which rearranges user space along the lines of the revised diagram on the left. The mmap area has been moved up to the top of the address space, and it now grows downward toward the heap. As a result, the bulk of the address space is preserved in a single, contiguous chunk which can be allocated to either the heap or mmap, as the application requires. 

As an added bonus, this organization reduces the amount of kernel memory required to hold each process's page tables, since the fragment at 0x40000000 is no longer present. 

There are a couple of disadvantages to this approach. One is that the stack area is rather more confined than it used to be. The actual size of the stack area is determined by the process's stack size resource limit, with a sizable cushion added, so problems should be rare. The other problem is that, apparently, a very small number of applications get confused by the new layout. Any application which is sensitive to how virtual memory is laid out is buggy to begin with; [according to Arjan van de Ven](<https://lwn.net/Articles/91848/>), the most common case is applications which store pointers in integer variables and then do the wrong thing when they see a "negative" value. 

The fact is that most users will never notice the change; for a demonstration, consider that Fedora kernels have been shipping with this patch for some time. Even a vanilla Fedora Core 1 system has it; a command like "`cat /proc/self/maps`" will show the new layout at work. The patch is currently part of the -mm kernel, and will probably find its way into the mainline before too long.
