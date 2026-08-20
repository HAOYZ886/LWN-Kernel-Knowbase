---
title: Linux support for ARM big.LITTLE
url: https://lwn.net/Articles/481055/
date: "February 15, 2012"
category: "Architectures-Arm; Scheduler-big.LITTLE; big.LITTLE"
author: "February 15, 2012 This article was contributed by Nicolas Pitre"
---

February 15, 2012

This article was contributed by Nicolas Pitre

ARM Ltd recently announced the [big.LITTLE architecture](<http://www.arm.com/products/processors/technologies/bigLITTLEprocessing.php>) consisting of a twist on the SMP systems that we've all gotten accustomed to. Instead of having a bunch of identical CPU cores put together in a system, the big.LITTLE architecture is effectively pushing the concept further by pulling two different SMP systems together: one being a set of "big" and fast processors, the other one consisting of "little" and power-efficient processors. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In practice this means having a cluster of [Cortex-A15](<http://www.arm.com/products/processors/cortex-a/cortex-a15.php>) cores, a cluster of [Cortex-A7](<http://www.arm.com/products/processors/cortex-a/cortex-a7.php>) cores, and ensuring cache coherency between them. The advantage of such an arrangement is that it allows for significant power saving when processes that don't require the full performance of the Cortex-A15 are executed on the Cortex-A7 instead. This way, non-interactive background operation, or streaming multimedia decoding, can be run on the A7 cluster for power efficiency, while sudden screen refreshes and similar bursty operations can be run on the A15 cluster to improve responsiveness and interactivity. 

Then, how to support this in Linux? This is not as trivial as it may seem initially. Let's suppose we have a system comprising a cluster of four A15 cores and a cluster of four A7 cores. The naive approach would suggest making the eight cores visible to the kernel and letting the scheduler do its job just like with any other SMP system. But here's the catch: SMP means Symmetric Multi-Processing, and in the big.LITTLE case the cores aren't symmetric between clusters. 

The Linux scheduler expects all available CPUs to have the same performance characteristics. For example, there are provisions in the scheduler to deal with things like hyperthreading, but this is still an attribute which is normally available on all CPUs in a given system. Here we're purposely putting together a couple of CPUs with significant performance/power characteristic discrepancies in the same system, and we expect the kernel to make the optimal usage of them at all times, considering that we want to get the best user experience together with the lowest possible battery consumption. 

So, what should be done? Many questions come to mind: 

  * Is it OK to reserve the A15 cluster just for interactive tasks and the A7 cluster for background tasks? 
  * What if the interactive tasks are sufficiently light to be processed by the small cores at all times? 
  * What about those background tasks that the user interface is actually waiting after? 
  * How to determine if a task using 100% CPU on a small core should be migrated to a fast core instead, or left on the small core because it is not critical enough to justify the increased power usage? 
  * Should the scheduler auto-tune its behavior, or should user-space policies influence it? 
  * If the latter, what would the interface look like to be useful and sufficiently future-proof? 

Linaro started an [initiative](<https://blueprints.launchpad.net/linux-linaro/+spec/linaro-kernel-q112-baby-steps-to-big-little-mp>) during the most recent [Linaro Connect](<http://connect.linaro.org/events/event/lcq1-12/>) to investigate this problem. It will require a high degree of collaboration with the upstream scheduler maintainers and a good amount of discussion. And given past history, we know that scheduler changes cannot happen overnight... unless your name is Ingo that is. Therefore, it is safe to assume that this will take a significant amount of time. 

Silicon vendors and portable device makers are not going to wait though. Chips implementing the big.LITTLE architecture will appear on the market in one form or another, way before a full heterogeneous multi-processor aware scheduler is available. An interim solution is therefore needed soon. So let's put aside the scheduler for the time being. 

ARM Ltd has produced a prototype software solution consisting of a small hypervisor using the virtualization extensions of the Cortex-A15 and Cortex-A7 to make both clusters appear to the underlying operating system as if there was only one Cortex-A15 cluster. Because the cores within a given cluster are still symmetric, all the assumptions built into the current scheduler still hold. With a single call, the hypervisor can atomically suspend execution of the whole system, migrate the CPU states from one cluster to the other, and resume system execution on the other cluster without the underlying operating system being aware of the change; just as if nothing has happened. 

Taking the example above, Linux would see only four Cortex-A15 CPUs at all times. When a switch is initiated, the registers for each of the 4 CPUs in cluster A are transferred to corresponding CPUs in cluster B, interrupts are rerouted to the CPUs in cluster B, then CPUs in cluster B are resumed exactly where cluster A was interrupted, and, finally, the CPUs in cluster A are powered off. And vice versa for switching back to the original cluster. Therefore, if there are eight CPU cores in the system, only four of them are visible to the operating system at all times. The only visible difference is the observable execution speed, and of course the corresponding change in power consumption when a cluster switch occurs. Some latency is implied by the actual switch of course, but that should be very small and imperceptible by the user. 

This solution has advantages such as providing a mechanism which should work for any operating system targeting a Cortex-A15 without modifications to that operating system. It is therefore OS-independent and easy to integrate. However, it brings a certain level of complexity such as the need to virtualize all the differences between the A15 and the A7. While those CPU cores are functionally equivalent, they may differ in implementation details such as cache topology. That would force every cache maintenance operation to be trapped by the hypervisor and translated into equivalent operations on the actual CPU core when the running core is not the one that the operating system thinks is running. 

Another disadvantage is the overhead of saving and restoring the full CPU state because, by virtue of being OS-independent, the hypervisor code may not know what part of the CPU is actually being actively used by the OS. The hypervisor could trap everything to be able to know what is being touched allowing partial context transfers, but that would be yet more complexity for a dubious gain. After all, the kernel already knows what is being used in the CPU, and it can deal with differing cache topologies natively, etc. So why not implement this switcher support directly in the kernel given that we _can_ modify Linux and do better? 

In fact that's exactly what we are doing i.e. take the ARM Ltd BSD licensed [switcher code](<http://git.linaro.org/gitweb?p=arm/big.LITTLE/switcher.git;a=summary>) and use it as a reference to actually put the switcher functionality directly in the kernel. This way, we can get away with much less support from the hypervisor code and improve switching performances by not having to trap any cache maintenance instructions, by limiting the CPU context transfer only to the minimum set of active registers, and by sharing the same address space with the kernel. 

We can implement this switcher by modeling its functionality as a CPU speed change, and therefore expose it via a [cpufreq](<http://www.ibm.com/developerworks/linux/library/l-cpufreq-1/index.html>) driver. This way, contrary to the reference code from ARM Ltd which is limited to a whole cluster switch, we can easily pair each of the A15 cores with one of the A7 cores, and have each of those CPU pairs appear as a single pseudo CPU with the ability to change its performance level via cpufreq. And because the cpufreq governors are already available and understood by existing distributions, including Android, we therefore have a straightforward solution with a fast time-to-market for the big.LITTLE architecture that shouldn't cause any controversy. 

Obviously the "switcher" as we call it is not replacing the ultimate goal of exposing all the cores to the kernel and letting the scheduler make the right decisions. But it is nevertheless a nice self-contained interim solution that will allow pretty good usage of the big.LITTLE architecture while removing the pressure to come up with scheduler changes quickly.
