---
title: Introducing io_uring_spawn
url: https://lwn.net/Articles/908268/
date: "September 20, 2022"
category: io uring
author: "By Jake Edge September 20, 2022 LPC"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
September 20, 2022

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2022-Linux_Plumbers_Conference>)

The traditional mechanism for launching a program in a new process on Unix systems—forking and execing—has been with us for decades, but it is not really the most efficient of operations. Various alternatives have been tried along the way but have not supplanted the traditional approach. A new mechanism created by Josh Triplett adds process creation to the [io_uring asynchronous I/O API](<https://lwn.net/Articles/776703/>) and shows great promise; he came to the [2022 Linux Plumbers Conference](<https://lpc.events/>) (LPC) to introduce io_uring_spawn. 

Triplett works in a variety of areas these days, much of it using the Rust language, though he has also been working on the kernel some of late. He is currently working on build systems as well. Build systems are notorious for spawning lots of processes as part of their job, "so I care about launching processes quickly". As with others at this year's LPC, Triplett said that he was happy to see a return to in-person conferences. 

#### Spawning a process

He began with a description of how a Unix process gets started. There are a number of setup tasks that need to be handled before a new process gets executed; these are things like setting up file descriptors and redirection, setting process priority and CPU affinities, dealing with signals and masks, setting user and group IDs, handling namespaces, and so on. There needs to be code to do that setup, but where does it come from? 

[ ![\[Josh Triplett\]](https://static.lwn.net/images/2022/lpc-triplett-sm.png) ](<https://lwn.net/Articles/908815/>)

The setup code for the traditional [`fork()`](<https://man7.org/linux/man-pages/man2/fork.2.html>) and exec (e.g. [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>)) approach must be placed in the existing process. `fork()` duplicates the current process into a second process that is a copy-on-write version of the original; it does not copy the memory of the process, just the page metadata. Then exec "will promptly throw away that copy and replace it with a new program; if that sounds slightly wasteful, that's because it's slightly wasteful". 

He wanted to measure how expensive the fork-and-exec mechanism actually is; he described the benchmarking tool that he would be using for the tests in the talk. The test creates a pipe, reads the start time from the system, then spawns a child process using whichever mechanism is under test; the exec of the child uses a path search to find the binary in order to make it more realistic. The child simply writes the end time to the pipe and exits, using a small bit of optimized assembly code. 

The parent blocks waiting to read the end time from the pipe, then calculates the time spent. It does that 2000 times and reports the lowest value; the minimum is used because anything higher than that is in some fashion overhead that he wants to eliminate from his comparison. The intent is to capture the amount time between the start of the spawn operation and the first instruction in the new process. Using that, he found that `fork()` and exec used 52µs on his laptop. 

But that is just a baseline for a process without much memory. If the parent allocates 1GB, the cost goes up a little bit to 56.4µs. But it turns out that Linux has some "clever optimizations" to handle the common case where processes allocate a lot more memory than they actually use. If the parent process touches all of the 1GB that it allocated, things get much worse, though: over 7500µs (or 7.5ms) 

There are more problems with `fork()` beyond just performance, however. For example, "`fork()` interacts _really_ badly with threads"; any locks held by other threads will remain held in the child forever. The `fork()` only copies the current thread, but copies all of the memory, which could contain locked locks; calling almost any C library function could then simply deadlock, he said. 

There is a list of safe C library functions in the [`signal-safety` man page](<https://man7.org/linux/man-pages/man7/signal-safety.7.html>), but it lacks some expected functions such as [`chroot()`](<https://man7.org/linux/man-pages/man2/chroot.2.html>) and [`setpriority()`](<https://man7.org/linux/man-pages/man3/setpriority.3p.html>). So if you fork a multi-threaded process, you cannot safely change its root directory or set its priority; "let alone things like setting up namespaces", he said. Using `fork()` is just not a good option for multi-threaded code. 

"As long as we are talking about things that are terribly broken, let's talk about [`vfork()`](<https://man7.org/linux/man-pages/man2/vfork.2.html>)". Unlike `fork()`, `vfork()` does not copy the current process to the child process, instead it "borrows" the current process. It is, effectively creating an unsynchronized thread as the child, which runs using the same stack as the parent. 

After the `vfork()` call, the child can do almost nothing: it can exec or exit—"that's the entire list". It cannot write to any memory, including the local stack (except for single process ID value), and cannot return or call anything. He rhetorically wondered what happens if the child happens to receive a signal; that is "among the many things that can go horribly wrong". Meanwhile, it does not provide any means for doing the kind of setup that might be needed for a new process. 

So given that `vfork()` is broken, he said, "let's at least hope it's broken and fast". His benchmark shows that it is, in fact, fast, coming in at 31.5µs for the base test and there is only a tiny increase, to 31.9µs, for allocating and accessing 1GB. That makes sense because `vfork()` is not copying any of the process memory or metadata. 

Another option is [`posix_spawn()`](<https://man7.org/linux/man-pages/man3/posix_spawn.3.html>), which is kind of like a safer `vfork()` that combines process creation and exec all in one call. It does provide a set of parameters to create a new process with certain attributes, but programmers are limited to that set; if there are other setup options needed, `posix_spawn()` is not the right choice. It has performance in between `vfork()` and `fork()` (44.5µs base); as with `vfork()`, there is almost no penalty for allocating and accessing 1GB (44.9µs). 

The main need for a copy of the original process is to have a place where the configuration code for the new process can live. `fork()`, `vfork()`, and `posix_spawn()` allow varying amounts of configuration for the new process. But a more recent kernel development provides even more flexibility—and vastly better performance—than any of the other options. 

#### Enter io_uring

The io_uring facility provides a mechanism for user space to communicate with the kernel through two shared-memory ring buffers, one for submission and another for completion. It is similar to the [NVMe](<https://en.wikipedia.org/wiki/NVM_Express>) and [Virtio](<https://www.linux-kvm.org/page/Virtio>) protocols. Io_uring avoids the overhead of entering and exiting the kernel for every operation as a system-call-based approach would require; that's always been a benefit, but the additional cost imposed by speculative-execution mitigations makes avoiding system calls even more attractive. 

In addition, io_uring supports linked operations, so that the outcome of one operation can affect further operations in the submission queue. For example, a read operation might depend on a successful file-open operation. These links form chains in the submission queue, which serialize the operations in each chain. There are two kinds of links that can be established, a regular link where the next operation (and any subsequent operations in the chain) will not be performed if the current operation fails, or a "hard" link that will continue performing operations in the chain, even when there are failures. 

So, he asked, what if we used io_uring for process setup and launch? A ring of linked operations to all be performed by the kernel—in the kernel—could take care of the process configuration and then launch the new process. When the new process is ready, the kernel does not need to return to the user-space process that initiated the creation, thus it does not need to throw away a bunch of stuff that it had to copy as it would with `fork()`. 

To that end, he has added two new io_uring operations. `IORING_OP_CLONE` creates a new task, then runs a series of linked operations in that new task in order to configure it; `IORING_OP_EXEC` will exec a new program in the task and if that is successful it skips any subsequent operations in the ring. The two operations are independent, one can clone without doing an exec, or replace the current program by doing an exec without first performing a clone. But they are meant to be used together. 

If the chain following an `IORING_OP_CLONE` runs out of ring operations to perform, the process is killed with `SIGKILL` since there is nothing for that process to do at that point. It is important to stop processing any further operations after a successful exec, Triplett said, or a trivial security hole can be created; if there are operations on the ring after an exec of a setuid-root program, for example, they would be performed with elevated privileges. If the exec operation fails, though, hard links will still be processed; doing a path search for an executable is likely to result in several failures of this sort, for example. 

#### Beyond performance

There are advantages to this beyond just performance. Since there is no user space involved, the mechanism bypasses the C library wrappers and avoids the "user-space complexity, and it is considerable", especially for `vfork()`. Meanwhile, the problems with spawning from multi-threaded programs largely disappear. He showed a snippet of code that uses [liburing](<https://github.com/axboe/liburing>) to demonstrate that the combination "makes this remarkably simple to do". That example can be seen on slide 55 of his [slides](<https://lpc.events/event/16/contributions/1213/attachments/1012/1945/io-uring-spawn.pdf>), or in the [YouTube video](<https://youtu.be/_h-kV8AYYqM?t=4104>) of the talk. 

Because he had been touting the non-performance benefits of using io_uring to spawn new programs, perhaps some in the audience might be thinking that the performance was not particularly good, he said. That is emphatically not the case; "actually it turns out that it is faster across the board". What he is calling "io_uring_spawn" took 29.5µs in the base case, 30.2µs with 1GB allocated, and 28.6µs with 1GB allocated and accessed. 

That is 6-10% faster than `vfork()` and 30+% faster than `posix_spawn()`, while being much safer, easier to use, and allowing arbitrary configuration of the new process. "This is the fastest available way to launch a process now." 

"Now" should perhaps be in quotes, at least the moment, as he is working with io_uring creator Jens Axboe to get the feature upstream. Triplett still needs to clean up the code some and they need to decide where the right stopping point, between making it faster and getting it upstream, lies. The development of the feature is just getting started at this point, he said; there are multiple opportunities for optimization that should provide even better performance down the road. 

#### Next steps

He has some plans for further work, naturally, including implementing `posix_spawn()` using io_uring_spawn. That way, existing applications could get a 30% boost on their spawning speed for free. He and Axboe are working on a pre-spawned process-pool feature that would be useful for applications that will be spawning at least a few processes over their lifetime. The pool would contain "warmed up" processes that could be quickly used for an exec operation. 

The clone operation could also be optimized further, Triplett thinks. Right now, it uses all of the same code that other kernel clone operations use, but a more-specialized version may be in order; it may make sense to reduce the amount of user-space context that is being created since it is about to be thrown away anyway. He would also like to experiment with creating a process from scratch, rather than copying various pieces from the existing process; the io_uring pre-registered file descriptors could be used to initialize the file table, for example. 

Triplett closed his talk with a shout-out to Axboe "who has been incredibly enthusiastic about this". Axboe has been "chomping at the bit to poke at it and make it faster". At some point, Triplett had to push back so that he had time to write the talk; since that is now complete, he expects to get right back into improving io_uring_spawn. He is currently being [sponsored on GitHub](<https://github.com/sponsors/joshtriplett>) for io_uring_spawn, Rust, and build systems; he encouraged anyone interested in this work to get in touch. 

After a loud round of applause, he took questions. Christian Brauner said that systemd is planning to use io_uring more and this would fit in well; he wondered if there was a plan to add support in io_uring for additional system calls that would be needed for configuring processes. Triplett said that he was in favor of adding any system call needed to io_uring, but he is not the one who makes that decision. "I will happily go on record as saying I would love to see a kernel that has exactly two system calls: `io_uring_setup()` and `io_uring_submit()`." 

Kees Cook asked how Triplett envisioned io_uring_spawn interacting with Linux security modules (LSMs) and [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>). Triplett said that it would work as well as io_uring works with those security technologies today; if the hooks are there and do not slow down io_uring, he would expect them to keep working. Paul Moore noted some of the [friction](<https://lwn.net/Articles/902466/>) that occurred when the command-passthrough feature was added to io_uring; he asked that the LSM mailing list be copied on the patches. 

A remote attendee asked about io_uring support for [Checkpoint/Restore in Userspace](<https://criu.org/Main_Page>) (CRIU). Axboe said that there is currently no way for CRIU to gather up in-progress io_uring buffers so that they can be restored; that is not a problem specific to io_uring_spawn, though, Triplett said. Brauner noted that there is a Google Summer of Code project to add support for io_uring to CRIU if anyone is interested in working on it. 

Brauner asked about whether the benchmarks included the time needed to set up the ring buffers; Triplett said that they did not, but it is something that is on his radar for the upstream submission. It is not likely to be a big win (or, maybe, a win at all) for a process that is just spawning one other process, but for programs like `make`, which spawn a huge number of other programs, the ring-buffer-creation overhead will fade into the noise. 

[I would like to thank LWN subscribers for supporting my travel to Dublin for Linux Plumbers Conference.]
