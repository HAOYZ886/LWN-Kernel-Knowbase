---
title: Too many threads
url: https://lwn.net/Articles/229873/
date: "April 10, 2007"
category: Kernel threads; Scalability
author: 
---

Part of the fun of working with truly large machines is that one gets to discover new scalability surprises before anybody else. So the SGI folks often have more fun than many of the rest of us. Their latest discovery has to do with the number of kernel threads which, on a 4096-processor system, leads to some interesting kernel behavior. 

To begin, they found out that they could not even boot a kernel with the default configuration. Linux systems normally have a limit of 32768 active processes at any given time. Anybody who has run "ps" will have noted that kernel threads are taking up an increasing number of those slots; your editor's single-processor desktop is running 39 of them. In fact, there are now enough kernel threads on a typical system that they will fill that entire space - and more - on a 4096-CPU machine. This problem is relatively easy to take care of by raising the limit on the number of processes. But it gets more interesting from there. 

The init process is the parent of last resort for every other process on the system, including kernel threads. So, on a big system, init has a _lot_ of child processes. These children live on a big linked list; that list must be searched by various functions, including the variants of `wait()`. If the process being searched for is toward the end of the list, that search can take a long time. Since (1) most kernel threads are long-lived, and (2) new processes are put at the end of the list, chances are that a search will, indeed, be looking for a process at the end. 

Then, for the ultimate in fun, load a module into the kernel. The module loading process calls `stop_machine_run()` when the new module is being linked in; this function creates a high-priority kernel thread for each processor on the system. That thread will grab its assigned CPU and simply sit there until told to exit; while all CPUs are locked up in this way the linking process can be performed. Calling a function like `stop_machine_run()` is a somewhat antisocial act in the best of times. But, in the 4096-processor system, `stop_machine_run()` will create 4096 threads, each of which goes on the end of init's child list, and each of which must be searched for when the time comes to clean it up. The result is a system which simply stops for an extended period of time. 

One could argue that people with systems that large simply should not load modules, but there is a possibility of pushback from the user community. So other solutions need to be found. Robin Holt's [problem report](<https://lwn.net/Articles/229876/>) included a simple patch which moves exiting processes to the beginning of the child list. This change solves the immediate problem by making searches for those children find them without having to iterate through all of the long-lived processes which are not going anywhere. 

Linus had [a couple of alternatives](<https://lwn.net/Articles/229877/>). One was to create a separate list for zombie processes, eliminating that search altogether. Another was to stop making kernel threads be children of the init process since they have little to do with user space in any case. But some developers feel that the real solution might be to start cutting back on the number of kernel threads. 

The biggest culprit for kernel thread creation will certainly be workqueues, which, by default, create one thread for every CPU on the system. There are situations which can benefit from multiple threads and CPU locality, but there are undoubtedly many places where all of those threads are not needed. Cleaning them up would help to solve some of the scalability issues; as an added bonus it would remove some of the clutter from ps listings. 

In many cases, a workqueue may not be necessary at all. Instead, kernel subsystems could just use the "generic" keventd workqueue (which runs as the `events/`n threads). There are some issues with using keventd, including indeterminate latency and a small possibility of deadlocks, but, for many situations, it may work well enough. 

In other cases, using a thread makes sense. Tasks involving long delays are one example; running a function with multi-second delays in `keventd` is considered impolite. Work requiring complicated context also benefits from its own thread. But, in a number of cases, those threads need not be created until there is actually some work to be done. A quick ps run on most systems will show threads related to error handling, asynchronous I/O, bluetooth, and more. In the current scheme, they are created at boot (or module load) time and many of them may never do any real work before the system shuts down. Thread creation is cheap, so many of these threads could be created on demand when they are needed. 

There are probably some real improvements to be made in this area; all that's needed is somebody with the time and motivation to do the work. In the mean time, those of you with 4096-way systems may need to apply a patch or two.
