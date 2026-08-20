---
title: (Nearly) full tickless operation in 3.10
url: https://lwn.net/Articles/549580/
date: "May 8, 2013"
category: "Dynamic tick; Read-copy-update"
author: "By Jonathan Corbet May 8, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 8, 2013

On a typical Linux system, each running CPU will be diverted between 100 and 1000 times each second by the periodic timer interrupt. That interrupt is the CPU's cue to reconsider which process should be running, catch up with read-copy-update (RCU) callbacks, and generally handle any necessary housekeeping. This periodic "tick" can be reasonably compared to the infamous big kernel lock (BKL): it is convenient to have around, but it also has an effect on performance that makes developers wish to abolish it. The key difference might be that getting rid of the timer tick has taken rather longer than was required to eliminate the BKL. The 3.10 kernel will take an important step in that direction, though, with the addition of the "full NOHZ" mode — but a lot of limitations still apply. 

Linux has had a partial solution to the timer tick problem for years in the form of the `CONFIG_NO_HZ` configuration option. If that option is set, the timer tick will be turned off, but only when the CPU is idle. This mode improves the situation considerably; it allows idle CPUs to stay in deep sleep states, reducing power use. Systems with virtualized guests also benefit, since, otherwise, each guest would be servicing timer interrupts when it should be doing nothing. In short: disabling the timer tick when the processor is idle makes enough sense that most distributions do it by default. 

Indeed, given that letting sleeping CPUs lie is generally a good policy, one might wonder why this behavior is optional at all. The answer is that it increases the cost of moving into and out of the idle state, (very) slightly increasing the time it takes to get an idle CPU back to work. That cost may be considered excessive in highly latency-sensitive environments. For everybody else, disabling the timer tick for idle CPUs is almost certainly the right thing to do; for battery-powered systems that is doubly true. 

The next step — disabling the tick for non-idle processors — is a lot more work with a smaller reward, so it is not surprising that it has taken a while to come about. Frederic Weisbecker finally [took up the challenge](<https://lwn.net/Articles/420544/>) in 2010; after a lot of changes and help by others (Paul McKenney made [some significant RCU changes](<https://lwn.net/Articles/522262/>), for example), this work has been pulled into the 3.10 kernel. 

In 3.10, the `CONFIG_NO_HZ` option has been replaced by a three-way choice: 

  * **`CONFIG_HZ_PERIODIC`** is the old-style mode wherein the timer tick runs at all times. 
  * **`CONFIG_NO_HZ_IDLE`** (the default setting) will cause the tick to be disabled at idle, the way setting `CONFIG_NO_HZ` did in earlier kernels. 
  * **`CONFIG_NO_HZ_FULL`** will enable the "full" tickless mode. 

The build-system code has been set up so that "`make oldconfig`" on 3.10 should yield a configuration that matches the previous setting of `CONFIG_NO_HZ` with no intervention required. Full tickless mode defaults to off; selecting that mode will enable tasks to run without the timer tick, but there are a number of things to be aware of. 

Among those are the requirement that the CPUs available for running without a timer tick must be designated at boot time using the `nohz_full=` command-line parameter. The boot CPU cannot run in this mode — at least one CPU needs to continue to receive interrupts and do the necessary housekeeping. The `CONFIG_NO_HZ_FULL_ALL` configuration option causes all CPUs (other than the boot CPU) to run in the full tickless mode by default; it can still be overridden with `nohz_full=`, though. The set of full tickless CPUs cannot be changed after boot; the amount of work required to make that possible would be large, and there does not seem to be a pressing need for this ability. 

Even then, a running CPU will only disable the timer tick if there is a _single_ runnable process on that CPU. As soon as a second process appears, the tick is needed so that the scheduler can make the necessary time-slice decisions. And even with a single runnable process, it is not technically tickless, since the timer tick still needs to happen at least once per second to keep the scheduler happy. But dropping from as high as 1000Hz to 1Hz is obviously a significant improvement. Response-time jitter due to timer interrupts will be nearly eliminated, and, [according to Ingo Molnar](<https://lwn.net/Articles/549592/>), as much as 1% of the CPU's time will be saved. 

There are workloads out there that will benefit significantly from those improvements. High-performance computing (HPC) and realtime are obvious candidates; in both cases, dedicating a CPU to a single task is a fairly common tactic already. But, in an era where even phones have quad-core processors, having a single runnable process on a given CPU is not an uncommon situation. 

There are a lot more details to making full tickless operation work properly; setting up a system to use this feature requires a fair amount of fiddling at this time. At a minimum, the administrator should make extensive use of CPU affinities to keep unwanted processes (including kernel threads) off the relevant processors. Some RCU configuration is required as well; see [Documentation/timers/NO_HZ.txt](<https://lwn.net/Articles/549593/>) for lots of details on the various options. 

Full tickless operation, as seen in 3.10, is clearly a significant step forward, but, equally clearly, this project is not yet complete. There is a fair amount of detail work to be done, including making the feature work on 32-bit processors (a patch exists), getting rid of that final once-per-second tick, mitigating some unfortunate side effects on the scheduler's statistics and load balancing, and fixing the inevitable bugs. This is a large and invasive change to how the core kernel works; there will almost certainly be some surprising behaviors that emerge once the tickless mode starts to get wider testing. 

The biggest item on the "to do" list, though, must surely be getting rid of the single-runnable-process requirement. Just in case the developers involved did not already feel that way, Linus [made his opinion on the matter clear](<https://lwn.net/Articles/549595/>): 

So as long as the NOHZ is for HPC-style loads, then quite frankly, I don't feel it is worth it. The _only_ thing that makes it worth it is that "future plans" part where it would actually help real loads. 

So, chances are, this limitation will be removed from the tickless implementation in some future development cycle, along with the other various rough edges. In the meantime, the 3.10 kernel will contain a significant step forward in the evolution of the core Linux kernel: the partial removal of a source of latency and overhead that has been there since the very first kernel release. Not even the big kernel lock endured anywhere near that long.
