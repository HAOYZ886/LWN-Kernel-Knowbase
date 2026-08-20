---
title: "The Ottawa Kernel Summit, Day One"
url: https://lwn.net/Articles/3327/
date: "June 24, 2002"
category: Kernel Summit
author: 
---

The first day of the Linux Kernel Developers' Summit has just finished in Ottawa. Almost 80 kernel hackers have gathered here to discuss issues that must be resolved in the 2.5 development series. This article covers the discussion from the first day of the summit. 

## Hammer port

This talk was mostly a status report on the AMD Hammer (x86-64) port. AMD, as one of the sponsors of the kernel summit, was given a slot of time to talk about this architecture and what has been done to make Linux work with this processor. 

Hammer is the extension of the old x86 architecture to 64 bits. It has been done in the most simple way possible; only two new instructions have been added. It is meant to be an easy path to 64-bit computing. 

Numerous details of the Hammer port were presented. Some of the more interesting ones: 

  * The stack remains two pages long, despite the fact that 64-bit code requires more stack space (more data space in general, really). To make this work, a separate, 16KB interrupt stack has been created for each IRQ line. 

  * Certain system calls (i.e. `gettimeofday()`) have been implemented with "vsyscalls." They run in user space, but live in a special memory region at the end of user space. 

  * Hammer uses a four-level page table. The Linux kernel only uses three levels, so the port authors made the fourth level global, shared by all processes. The page size remains 4KB (any other size presents application compatibility problems). The current virtual address space is 512GB, which is "enough for now." Processes running in 32-bit mode have a 3.5GB address space. A 512GB segment has been set aside just for kernel memory obtained with `vmalloc`. The kernel is also able to map all memory directly into kernel space, so there is no "high memory" on this architecture. 

Overall, this port looks like a nice way to run Linux - your author wishes he had one of these systems. 

## Kernel parameters

"Kernel parameters" are just ways of controlling how the kernel operates. Typically, they can be specified at boot time (on the command line), at module load time, or at run time through `/proc`, a system call, or some other interface. Each way of handling parameters requires a different bit of code, so many kernel subsystems do not implement all the different ways of setting parameters. This session looked at two different approaches for unifying the handling of kernel parameters. 

Rusty Russell presented his scheme for handling kernel parameters. It involves replacing the `MODULE_PARM` macro with a new `PARAM` macro which would set up a parameter for the boot command line, module insertion, and `/proc`. Unlike `MODULE_PARM`, the new scheme is type safe (i.e. the compiler catches type errors). It is also based on a callback mechanism which makes it easy for a module to define new parameter types if need be. 

The other approach was presented by Patrick Mochel. As part of his ongoing driver model work, he implemented "driverfs," the driver filesystem. driverfs is a virtual filesystem initially developed as a debugging tool; it is a convenient way of looking at the structure of the device tree. The facilities provided by driverfs are similar to those provided by Rusty's scheme; driverfs, too, is type-safe and easy to set up. It is also, for now, limited to device drivers only; the question is whether it should be expanded to handle kernel parameters in general. 

While no conclusions were drawn in the session, driverfs looks (to your author) like it is here to stay, since it so nicely incorporates the system's structure in the filesystem it creates. It just needs to be expanded to cover all of the other types of parameters in the system (i.e. VM, networking, etc.) that are not directly associated with devices. A merger with some variant Rusty's scheme would help in that regard, and seems likely at some point. 

Linus pointed out that he would like to see a single, integrated filesystem that includes all such information. Thus, for example, the "fsfs" that provides filesystem information can include links to the underlying devices holding those filesystems. Module information can link to the associated driver and devices. Such a filesystem, he said, makes the linkages in the system visible and explicit. 

Eventually, `/proc` could be phased out in favor of a driverfs-like system, with a "one file, one value" rule. This change really has to begin in 2.5, since `/proc` has to be supported for at least one more stable kernel cycle. Expect to see more activity in this area in the near future. 

## Living with modules

The session on "living with modules" was presented by Rusty Russell "and the angry ghost of Keith Owens." Rusty started with the claim that the only purpose for modules was adding hardware that you didn't have when you booted your kernel. Given that, he would not be entirely ill at ease with the removal of loadable modules entirely. But that, of course, is not to be. 

There are, according to Rusty, three classes of problems with loadable modules: 

  * implementation problems 
  * initialization problems 
  * removal problems 

The implementation problems are mostly poor interfaces and badly done subsystems - stuff that can be fixed. Rusty singled out the `inter_module` calls as being especially poorly done and impossible to use correctly (thus making Keith's ghost even angrier). 

Module linking could also be improved: Rusty set out to do that by moving the linking code from the `insmod` program to the kernel itself. The result: the amount of kernel code actually dropped. Many things get easier, it seems, when you don't have to deal with the differences between the kernel and user space environments. As an added bonus, he implemented the discarding of initialization code - something that has never quite been done for loadable modules. 

One thing Rusty did not yet implement was modversions (matching of data structures to allow binary modules to be loaded into more than one kernel version). Linus asked if anybody would be truly unhappy if modversions went away forever. There were few complaints: modversions is ugly, and kernel developers rarely use it. Alan Cox and the rest of the Red Hat crowd did complain, though, saying that customer support would be a nightmare if the kernel did not support modversions. The real-world need for modversions is probably strong enough that the feature will not go away. 

Moving on, Rusty got into module initialization problems. There are a few classes of these: 

  * Most initialization code assumes that it is being run at boot time, when there are no user processes running yet. So, for example, it is common to see a module set up a `/proc` interface before it is ready, internally, to handle accesses to that interface. This sort of race condition almost never hurts users, but it does exist. 

  * Error checking in module initialization code is often inadequate. Poor interfaces can be blamed for part of this. 

  * Recovering from errors at initialization time is almost impossible. Imagine a driver which creates two files in `/proc`. If something goes wrong between the creation of those two files, the first one must be removed. But the possibility exists that some user process has already opened the first file. There is no safe action at that point. 

There are a couple of possible solutions to the initialization issues. The first is to split every kernel registration interface into two phases: "reserve" and "use." The reserve functions would allocate the needed resources to the module, but would not make any interfaces available to the kernel or to user space. When all of the reservations have succceeded, the module can call the "use" functions (which will not fail) to safely make resources available to the rest of the system. 

The alternative is to add a module pointer to every registration interface. Any use of the registered resource would then increment the reference count on the module, preventing its premature removal. Even if the module initialization fails, the module will not be unloaded until the reference count drops to zero. These changes, along with an audit of the initialization code in every module, would make module loading safe. 

Module removal is a more difficult problem. Some of the issues here include: 

  * There is no way to force the removal of a module. Some modules, in fact, are not removable at all. 

  * The module removal operation itself can not return a failure status, since its return value is void. 

  * The biggest issue is the raft of race conditions that go along with module removal. It is _hard_ to know when it is truly safe to remove a module from the system. 

There are a few ways of dealing with the removal problems. One would be to add locking and reference counts to absolutely every operation that can involve a module. Everything which tries to increment a module reference count must check the status of the operation and not use the module if the increment fails (which can happen if the module is being deleted). The change is invasive, and the performance overhead of all that reference counting could be significant. 

The second option is a two-stage module removal process. The first stage ("cleanup") removes all interfaces to the module, guaranteeing that the module reference count will not increase. The cleanup step can fail, in which case the module will not be unloaded. Assuming a successful cleanup, the "destroy" phase actually removes the module from the system once it's safe to do so. 

"Once it's safe" is still not an easy thing to know, however. The kernel must wait until the reference count on the module drops to zero, of course. But, at the minimum, there is still a race condition here: the module must execute, at least, a "return" instruction after decrementing its reference count. It is conceivable that the kernel could remove the module before the return is executed, with unpredictable results. 

To get around this problem, the original cleanup/destroy patch waited for every processor on the system to schedule once; that would guarantee that no processor is still executing with the doomed module's code. At least, that worked until the preemptible kernel patch was merged. With a preemptible kernel, the only way to be truly safe is to wait until every process on the system has yielded the processor once. 

There are advantages to this system: it can be made to work safely, it is possible to force the unload of a module, and each module can handle its own reference counting. It is complex, however. 

The third approach is simpler: simply deprecate module removal. It would remain as an optional feature (it is, after all, most useful for debugging kernel code), but, by default, module loading would be forever. In the end, says Rusty, kernel modules do not take that much space and memory is cheap; module removal may be an optimization that we no longer need. There are some residual issues, such as old PCMCIA drivers that do not properly clean up after themselves if they are not removed, but as a whole this option is easy and makes a lot of code go away. There seemed to be a lot of sympathy for this approach in the room. No decisions were made, however. 

## Virtual memory

The VM session featured Rik van Riel and Andrea Arcangeli; it was moderated by Daniel Phillips. One might have expected this to be a contentious session, given the disagreements that have come up over the years, but it turned out not to be that way. Linux VM, it seems, may actually be approaching a relatively mature state. 

The first topic of discussion was Rik van Riel's reverse mapping VM patch (see [the January 24 LWN Kernel Page](</2002/0124/kernel.php3>)). Rik approached it as a controversial subject, pointing out its advantages but prepared for a discussion on whether, in the end, rmap justified the overhead it imposes. Andrea expressed concern over the extra costs (especially in the `fork()` system call), but agreed that rmap was needed in a number of situations. 

Linus cut much of the discussion short, however, by stating that rmap was almost certain to go into 2.5 at some point. He is more concerned with getting a set of small patches that he can merge into the kernel so that the remaining issues can be addressed. So there wasn't much point in talking further about rmap's advantages and disadvantages. 

The issues of accounting and out-of-memory handling remain. Interestingly, it is hard for the kernel to know when it is really out of memory; it is hard to keep a handle on how many pages could be freed now if they were needed. So heuristics must be used, and a balance must be found. If an "out of memory" situation is declared too soon, processes can be killed unnecessarily. If you wait too long, the system can deadlock. Andrea's current code errs on the side of killing processes, rather than risk deadlocks. 

An interesting source of VM problems is kmap entries. 32-bit systems with large amounts of memory must set aside most of the memory as "high memory," which has no direct mapping from kernel space. Before any kernel code can access high memory, it must temporarily map it into kernel space via a kmap entry. It turns out that there can be demands for large numbers of kmap entries, exhausting the pool of page table entries set aside for kmaps. Additionally, the kmap pool is protected by a spinlock; on large systems, contention for this spinlock can be a real performance drag. 

The solution appears to be creating per-process pools of kmap entries. This implementation would scale the number of entries along with the number of processes, and would eliminate the contention for the global lock. Look for an implementation from Andrea before too long. 

Other issues that were discussed briefly include: 

  * NUMA systems, which are increasing in importance. 

  * Soft page size - making pages larger through clustering, at least in some memory zones. Larger pages can reduce system overhead, but can bring internal fragmentation and system complexity. 

  * Very large physical pages on architectures that support them. 

  * Swap clustering - swapping adjacent memory pages to consecutive blocks on the disk. Reverse mapping will make clustering harder, since it looks at physical memory rather than virtual memory areas. Rik thinks that this problem can be worked around, however. 

  * Better reclamation of system data structures. For example, the kernel trims down the directory entry (dentry) cache when memory is tight, but dentries are released via a least-recently-used queue. If, instead, the system freed all dentries on a given page, it could free the page more easily. 
The overall picture of Linux VM is that there remains much work to do, but that the basic mechanism is in relatively good shape. Given that people have been complaining about VM in Linux for years, this is a most positive development. 

## Block I/O

Jens Axboe led a session on the block I/O subsystem which, of course, has been massively reworked in the 2.5 series. His talk concentrated on the issues that remain to be resolved. 

One of those issues is ordered writes and barriers. Journaling filesystems need, at times, to be sure that certain operations have completed before others can be started. Without write barriers, the transactional nature of the filesystem is lost. The infrastructure for write barriers is there now; what remains is to push the implementation down into the block drivers. For IDE drives, this will be done with cache flushes before and after the barrier. For SCSI drives, ordered tags can be used. That requires, however, that the SCSI layer use the generic tag code which has been implemented in the block layer; that work is in progress. 

Multipage I/O still raises issues. Many I/O operations generated by the system are large; it is vastly preferable to keep them together so that they can be handled efficiently by the hardware. The problem is that, sometimes, the hardware can not handle large requests. Hardware limitations can come into play, or the block device could be a virtual device (a RAID or LVM device) which must split the request anyway. 

One way of handling this problem is to split requests that turn out to be too large. But splitting is an ugly and inefficient process; it is best avoided. A better approach would be to involve the device driver in the construction of block I/O requests; a new interface would allow the requests to be built, page by page, with the driver telling the block layer when the request gets too large. 

Even then, though, it seems that splitting may be necessary in some situations. The remaining cases could probably be solved in a simple way, however; the offending request could just be resubmitted one sector at a time. This solution is slow, but it shouldn't be needed that often. 

Unlike the rest of the block I/O subsystem, I/O scheduling remains essentially unchanged since 2.4. The current elevator code works well in most situations, but one can always try to do better. Jens has been experimenting with a variation of the scheduler which would enforce an upper bound on the latency for any given request. The modified elevator can guarantee that any request will be executed within one second, with a 3% performance penalty. Lowering the deadline to 100ms raises the performance hit to 8%. Most people seemed to think that this penalty was acceptible. Future work could include prioritizing requests and "anticipatory scheduling" - delaying read requests slightly in the hope that they can be clustered with other requests. 

Finally, the task of removing buffer heads from the block I/O subsystem continues. 

## Tomorrow's sessions

The second day will include sessions on database performance, the HP kernel wishlist (presented by Bdale Garbee), the Loadable Security Module, asynchronous I/O, SCSI, and the overall goals for 2.6 and "improving the release engineering process." Stay tuned for our coverage of these discussions. 

(**Update** : that coverage may now be found on [this page](<https://lwn.net/Articles/3467/>)).
