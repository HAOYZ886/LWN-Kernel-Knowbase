---
title: Shrinking the scheduler
url: https://lwn.net/Articles/725376/
date: "June 14, 2017"
category: Embedded systems; Scheduler
author: "By Jonathan Corbet June 14, 2017"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 14, 2017

The ups and downs of patching the kernel to wedge Linux into tiny systems has been debated numerous times over the years, most recently in the context of Nicolas Pitre's [alternative TTY layer patches](<https://lwn.net/Articles/721074/>) posted in April. Pitre is driving the debate again, this time by [trying to shrink the kernel's CPU scheduler](<https://lwn.net/Articles/724768/>). In the process, he has exposed a couple of areas of fundamental disagreement on the value of this kind of work. 

Pitre's goal is to make it possible to run a system based on a Linux kernel on a processor with as little as 256KB of memory. Doing so requires more than just making code and data structures smaller; he has to simply eliminate as much code as possible. With the TTY patches, he replaced the TTY subsystem outright with a much smaller (and less capable) alternative. His approach with the scheduler is different: the kernel's core scheduler remains when his patches are active, but a number of features, including the realtime and deadline scheduler classes, are compiled out. The resulting scheduler is 25% smaller at the cost of features that will almost certainly not be used on tiny systems anyway. 

As is often the case with "tinification" patches, the scheduler patches were given a chilly reception, with Ingo Molnar [rejecting](<https://lwn.net/Articles/725378/>) them outright. In the ensuing discussion, it became clear that there were two points of core disagreement on the value of these patches. 

Alan Cox [argued](<https://lwn.net/Articles/725379/>) that a kernel configured for such small systems isn't really Linux anymore: 

So once you've rewritten the tty layer, the device drivers, the VFS and removed most of the syscalls why even pretend it's Linux any more. It's something else, and that something else is totally architecturally incompatible with Linux. That's btw a good thing - trying to fit Linux directly into such a tiny device isn't sensible because the core assumptions you make about scalability are just totally different. 

His suggestion was to create an entirely new kernel, borrowing bits of Linux source where it helps. Pitre's [response](<https://lwn.net/Articles/725380/>) was that this approach has been tried many times without success. Special-purpose kernels tend to be small projects with few developers; they progress slowly, are poorly maintained, and suffer from a lack of code review. By making it possible to use much of the Linux kernel, including, importantly, its device drivers, Pitre hopes to pull together the tiny-systems community around a single, well-supported alternative. 

Molnar's [argument](<https://lwn.net/Articles/725382/>), instead, was that the value obtained by supporting such small systems is not worth the cost, as measured in code complexity. Moore's law may be slowing down, he said, but these systems will still become more capable over time. Given the time that passes between the application of a scheduler patch and its appearance in distributions and products — between two and five years — there may be no need for this work by the time it becomes widely available. Given that, he argued, it is far more important to reduce the complexity of the scheduler than to reduce its size: 

So while it obviously the "complexity vs. kernel size" trade-off will always be a judgment call, for the scheduler it's not really an open question what we need to do at this stage: we need to reduce complexity and #ifdef variants, not increase it. 

Pitre [disagreed](<https://lwn.net/Articles/725385/>) with Molnar on every point. The smallest systems, he said, will remain small for economic reasons: 

Your prediction is based on a false premise. There is simply no money to be made with IoT hardware, especially in the low end. Those little devices will be given away for free because it is in the service subscription that the money is. So the hardware has to, and will be, extremely cheap to produce. 

The need for extremely low power consumption, so that a system can run for months or years on a single battery, will also keep these systems small. With regard to the time lag for adoption of the changes, he pointed out that much of the Android-related code that has gone into the mainline has been merged years _after_ being deployed in products; the timing tends to be reversed in that part of the market. He also argued that his patches actually reduce the complexity of the scheduler code by factoring out the different scheduler classes and making it possible to remove them. 

The conversation did not progress much beyond that point. There was one important bit of progress, though: Molnar [agreed](<https://lwn.net/Articles/725388/>) that Pitre's code-movement patches make the scheduler more maintainable. He requested that those patches be posted on their own so that they can be merged; that, he said, ""should make future arguments easier"". Pitre [has obliged](<https://lwn.net/Articles/725389/>), but there has been no discussion of the new patches as of this writing. Should they be accepted, the remaining changes, which actually compile out those scheduler classes, should be quite small. 

It is clear that getting core kernel maintainers to accept the costs associated with supporting tiny systems will always be a hard sell. In this case, the tiny-systems community has a developer who is determined to get the job done, but who is also familiar with how kernel development works and is willing to make the changes needed to get his patches merged. That still doesn't guarantee success in this inherently difficult — if not quixotic — endeavor, but the odds this time around would seem to be better than with previous attempts which, as can be seen [here](<https://lwn.net/Articles/597529/>) or [here](<https://lwn.net/Articles/624258/>), have not always gone well.
