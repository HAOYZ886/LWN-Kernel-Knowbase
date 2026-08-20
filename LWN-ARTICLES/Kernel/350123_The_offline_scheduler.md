---
title: The offline scheduler
url: https://lwn.net/Articles/350123/
date: "September 2, 2009"
category: Scheduler
author: "By Jake Edge September 2, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
September 2, 2009

One of the primary functions of any kernel is to manage the CPU resources of the hardware that it is running on. A recent [patch](<http://lwn.net/Articles/348325/>), proposed by Raz Ben-Yehuda, would change that, by removing one or more CPUs out from under the kernel's control, so that processes could run, undisturbed, on those processors. The "offline scheduler", as Ben-Yehuda calls his patch, had some rough sailing in the initial reactions to the idea, but as the thread on linux-kernel evolved, kernel hackers stepped back and looked at the problems it is trying to solve--and came up with other potential solutions. 

The basic idea behind the offline scheduler is fairly straightforward: use the CPU hot-unplug facility to remove the processor from the system, but instead of halting the processor, allow other code to be run on it. Because the processor would not be participating in the various CPU synchronization schemes (RCU, spinlocks, etc.), nor would it be handling interrupts, it can completely devote its attention to the code that it is running. The idea is that code running on the offline processor would not suffer from any kernel-introduced latencies at all. 

The [core patch](<https://lwn.net/Articles/350305/>) is fairly small. It provides an interface to register a function to be called when a particular CPU is taken offline: 

```
int register_offsched(void (*offsched_callback)(void), int cpuid);
```

This registers a callback that will be made when the CPU with the given `cpuid` is taken offline (i.e. hot unplugged). Typically, a user would load a module that calls `register_offsched()`, then take the CPU offline which triggers the callback on the just-offlined CPU. When the processing completes, and the callback returns, the processor will then be halted. At that point, the CPU can be brought back online and returned to the kernel's control. 

The interface points to one of the problems that potential users of the offline scheduler have brought up: one can only run kernel-context, and not user-space, code using the facility. Because many of the applications that might benefit from having the full attention of a CPU are existing user-space programs, making the switch to in-kernel code is seen as problematic.

Ben-Yehuda [notes](<https://lwn.net/Articles/350306/>) that the isolated processor has ""access to every piece of memory in the system"" and the kernel would still have access to any memory that the isolated processor is using. He sees that as a benefit, but others, particularly Mike Galbraith, [see](<https://lwn.net/Articles/350307/>) it differently: 

I personally find the concept of injecting an RTOS into a general purpose OS with no isolation to be alien. Intriguing, but very very alien. 

One of the main problems that some kernel hackers see with the offline scheduler approach is that it bypasses Linux entirely. That is, of course, the entire point of the patch: devoting 100% of a CPU to a particular job. As Christoph Lameter [puts it](<https://lwn.net/Articles/350309/>): 

OFFSCHED takes the OS noise (interrupts, timers, RCU, cacheline stealing etc etc) out of certain processors. You cannot run an undisturbed piece of software on the OS right now. 

Peter Zijlstra, though, [sees](<http://lwn.net/Articles/350310/>) that as a major negative: ""Going around the kernel doesn't benefit anybody, least of all Linux."" There are existing ways to do the same thing, so adding one into the kernel adds no benefit, he says: 

So its the concept of running stuff on a CPU outside of Linux that I don't like. I mean, if you want that, go ahead and run RTLinux, RTAI, L4-Linux etc.. lots of special non-Linux hypervisor/exo-kernel like things around for you to run things outside Linux with. 

But, Ben-Yehuda sees multiple applications for processors dedicated to specific tasks. He [envisions](<https://lwn.net/Articles/350321/>) a different kind of system, which he calls a Service Oriented System (SOS), where the kernel is just one component, and if the kernel ""disturbs"" a specific service, it should be moved out of the way: 

What i am suggesting is merely a different approach of how to handle multiple core systems. instead of thinking in processes, threads and so on i am thinking in services. Why not take a processor and define this processor to do just firewalling ? encryption ? routing ? transmission ? video processing... and so on... 

Moving the kernel out of the way is not particularly popular with many kernel hackers. But the idea of completely dedicating a processor to a specific task is important to some users. In the high performance computing (HPC) world, multiple processors spend most of their time working on a single, typically number-crunching, task. Removing even minimal interruptions, those that perform scheduling and other kernel housekeeping tasks, leads to better overall performance. Essentially, those users want the convenience of Linux running on one CPU, while the rest of the system's CPUs are devoted to their particular application. 

After a somewhat heated digression about generally reducing latencies in the kernel, Andrew Morton [asked](<https://lwn.net/Articles/350330/>) for a problem statement: ""All I've seen is 'I want 100% access to a CPU'. That's not a problem statement - it's an implementation."" In answer, Chris Friesen [described](<https://lwn.net/Articles/350331/>) one possible application: 

In our case the problem statement was that we had an inherently single-threaded emulator app that we wanted to push as hard as absolutely possible. 

We gave it as close to a whole cpu as we could using cpu and irq affinity and we used message queues in shared memory to allow another cpu to handle I/O. In our case we still had kernel threads running on the app cpu, but if we'd had a straightforward way to avoid them we would have used it. 

That led Thomas Gleixner to [consider](<https://lwn.net/Articles/350338/>) an alternative approach. He restated the problem as: ""Run exactly one thread on a dedicated CPU w/o any disturbance by the scheduler tick."" Given that definition, he suggested a fairly simple approach: 

All you need is a way to tell the kernel that CPUx can switch off the scheduler tick when only one thread is running and that very thread is running in user space. Once another thread arrives on that CPU or the single thread enters the kernel for a blocking syscall the scheduler tick has to be restarted. 

Gregory Haskins then [suggested](<https://lwn.net/Articles/350340/>) modifying the FIFO scheduler class, or creating a new class with a higher priority, so that it disables the scheduler tick. That would incorporate Gleixner's idea into the existing scheduling framework. As might be guessed, there are still some details to work out on running a process without the scheduler tick, but Gleixner and others think it is something that can be done. 

The offline scheduler itself kind of fell by the wayside in the discussion. Ben-Yehuda, unsurprisingly, is still pushing his approach, but aside from the distaste expressed about circumventing the kernel, the inability to run user-space code is problematic. Gleixner was fairly [blunt](<https://lwn.net/Articles/350348/>) about it: 

I was talking about the problem that you cannot run an ordinary user space task on your offlined CPU. That's the main point where the design sucks. Having specialized programming environments which impose tight restrictions on the application programmer for no good reason are horrible. 

Others are also thinking about the problem, as a similar idea to Gleixner's was recently [posted](<http://lwn.net/Articles/350153/>) by Josh Triplett in an RFC to linux-kernel. Triplett's tiny patch simply disables the timer tick permanently as a demonstration of the gain in performance that can be achieved for CPU-bound processes. He notes that the overhead for the timer tick can be significant: 

On my system, the timer tick takes about 80us, every 1/HZ seconds; that represents a significant overhead. 80us out of every 1ms, for instance, means 8% overhead. Furthermore, the time taken varies, and the timer interrupts lead to jitter in the performance of the number crunching. 

Triplett warns that his patch is ""by no means represents a complete solution"" in that it breaks RCU, process accounting, and other things. But it does boot and can run his tests. He has fixes for some of those problems in progress, as well as an overall goal: ""I'd like to work towards a patch which really can kill off the timer tick, making the kernel entirely event-driven and removing the polling that occurs in the timer tick. I've reviewed everything the timer tick does, and every last bit of it could occur using an event-driven approach."" 

It is pretty unlikely that we will see the offline scheduler ever make it into the mainline, but the idea behind it has spawned some interesting discussions that may lead to a solution for those looking to eliminate kernel overhead on some CPUs. In many ways, it is another example of the perils of developing kernel code in isolation. Had Ben-Yehuda been working in the open, and looking for comments from the kernel community, he might have realized that his approach would not be acceptable--at least for the mainline--much sooner.
