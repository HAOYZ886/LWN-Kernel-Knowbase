---
title: Counting on the time stamp counter
url: https://lwn.net/Articles/209101/
date: "November 13, 2006"
category: TSC; Timekeeping
author: 
---

The time stamp counter (TSC) is a hardware feature found on a number of contemporary processors. The TSC is a special register which is simply incremented every clock cycle. Since the clock is the fundamental unit of time as seen by the processor, the TSC provides the highest-resolution timing information available for that processor. It can thus be used for a number of applications, such as measuring the exact time cost of specific instructions or operations. 

The TSC can also be read quickly (it is just a CPU register, after all), making it of interest for system timekeeping. There are a lot of applications which check the current time frequently, to the point that `gettimeofday()` is one of the most performance-critical system calls in Linux. By using the TSC to interpolate within the resolution of a coarser clock, the system can give accurate, high-resolution time without taking a lot of time in the process. 

That is the idea, anyway. In practice, the TSC turns out to be hard to use in this way. If the CPU frequency changes (as it will on CPUs which can vary their power consumption), the TSC rate will change as well. If the processor is halted (as can happen when it goes idle), the TSC may stop altogether. On multiprocessor systems, the TSCs on different processors may drift away from each other over time - leading to a situation where a process could read a time on one CPU, move to a second processor, and encounter a time earlier than the one it read on the first processor. 

These challenges notwithstanding, the Linux kernel tries to make the best use of the TSC possible. The code which deals with the TSC contains a number of checks to try to detect situations where TSC-based time might not be reliable. One of those checks, in particular, compares TSC time against the jiffies count, which is incremented by way of the timer tick. If, after ten seconds' worth of ticks, the number of TSC cycles seen differs from what would have been expected, the kernel concludes that the TSC is not stable and stops using it for time information. 

Interesting things happen when the dynamic tick patch is thrown into the mix. With dynamic ticks, the periodic timer interrupt is turned off whenever there's nothing to be done in the near future, allowing the processor to remain idle for longer and consume less power. Once something happens, however, the `jiffies` count must be updated to reflect the timer ticks which were missed - something which is generally done by obtaining the time from another source. At best, this series of events defeats the test which ensures that the TSC is operating in a stable manner; at worst, it can lead to corrupted system time. Not a good state of affairs. 

For this reason, the recently-updated [high-resolution timers and dynamic tick patch set](<http://lwn.net/Articles/209038/>) includes a change which disables use of the TSC. It seems that the high-resolution timers and dynamic tick features are incompatible with the TSC - and that people configuring kernels must choose between the two. Since the TSC does have real performance benefits, disabling it has predictably made some people unhappy, to the point that some would prefer to see the timer patches remain out of the kernel for now. 

In response to the objections, Ingo Molnar has [explained](<https://lwn.net/Articles/209167/>) things this way: 

We just observed that in the past 10 years no generally working TSC-based gettimeofday was written (and i wrote the first version of it for the Pentium, so the blame is on me too), and that we might be better off without it. If someone can pull off a working TSC-based gettimeofday() implementation then there's no objection from us. 

Ingo has also posted [a test program](<https://lwn.net/Articles/209168/>) which demonstrates that time inconsistencies on TSC-based systems are common - at least, when multiple processors are in use. 

Arjan van de Ven has [suggested](<https://lwn.net/Articles/209169/>) a "duct tape" solution which might work well enough "to keep the illusion alive." It involves setting up offsets and multipliers for each processor's TSC. Between the offsets (which could compensate for TSC drift between processors) and the multipliers (which adjust for frequency changes), some semblance of synchronized and accurate TSC-based time could be maintained - as long as the kernel is able to detect TSC-related events and adjust those values accordingly. No code which implements this idea has yet been posted, however. 

The conversation faded out with no real conclusion, though, near the end, Thomas Gleixner did [note](<https://lwn.net/Articles/209170/>) that the complete disabling of the TSC was "overkill." The preferred solution, which he is working on, is to keep the system from going into the dynamic tick mode if there is no other reliable timer available. Once that code has been posted, it may be possible to have the full set: high-resolution timers, dynamic ticks, and fast clocks using the TSC.
