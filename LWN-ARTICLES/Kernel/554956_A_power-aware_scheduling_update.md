---
title: "A power-aware scheduling update"
url: https://lwn.net/Articles/554956/
date: "June 19, 2013"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet June 19, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 19, 2013

Earlier this month, LWN [reported](<https://lwn.net/Articles/552885/>) on the "line in the sand" drawn by Ingo Molnar with regard to power-aware scheduling. The fragmentation of CPU power management responsibilities between the scheduler, CPU frequency governors, and CPUidle subsystem had to be replaced, he said, by an integrated solution that put power management decisions where the most information existed: in the scheduler itself. An energetic conversation followed from that decree, and a possible way forward is beginning to emerge. But the problem remains difficult. 

Putting the CPU scheduler in charge of CPU power management decisions has a certain elegance; the scheduler is arguably in the best position to know what the system's needs for processing power will be in the near future. But this idea immediately runs afoul of another trend in the kernel: actual power management decisions are moving away from the scheduler toward low-level hardware driver code. As Arjan van de Ven [noted](<https://plus.google.com/117091380454742934025/posts/2vEekAsG2QT>) in a May Google+ discussion, power management policies for Intel CPUs are being handled by CPU-specific code in recent kernels: 

We also, and I realize this might be controversial, combine the control algorithm with the cpu driver in one. The reality is that such control algorithms are CPU specific, the notion of a generic "for all cpus" governors is just outright flawed; hardware behavior is key to the algorithm in the first place. 

Arjan suggests that any discussion that is based on control of CPU frequencies and voltages misses an important point: current processors have a more complex notion of power management, and they vary considerably from one hardware generation to the next. The scheduler is not the right place for all that low-level information; instead, it belongs in low-level, hardware-specific code. 

There is, however, fairly widespread agreement that passing more information between the scheduler and the low-level power management code would be helpful. In particular, there is a fair amount of interest in better integration of the scheduler's load-balancing code (which decides how to distribute processes across the available CPUs) and the power management logic. The load balancer knows what the current needs are and can make some guesses about the near future; it makes sense that the same code could take part in deciding which CPU resources should be available to handle that load. 

Based on these thoughts and more, Morten Rasmussen has posted [a design proposal](<https://lwn.net/Articles/554981/>) for a reworked, power-aware scheduler. The current scheduler would be split into two separate modules: 

  1. The CPU scheduler, which is charged with making the best use of the CPU resources that are currently available to it. 

  2. The "power scheduler," which takes the responsibility of adjusting the currently available CPU resources to match the load seen by the CPU scheduler. 

The CPU scheduler will handle scheduling as it is done now. The power scheduler, instead, takes load information from the CPU scheduler and, if necessary, makes changes to the system's power configuration to better suit that load. These changes can include moving CPUs from one power state to another or idling (or waking) CPUs. The power scheduler would talk with the current frequency and idle drivers, but those drivers would remain as separate, hardware-dependent code. In this design, load balancing would remain with the CPU scheduler; it would not move to the power scheduler. 

Of course, there are plenty of problems to be solved beyond the simple implementation of the power scheduler and the definition of the interface with the CPU scheduler. The CPU scheduler still needs to learn how to deal with processors with varying computing capacities; the big.LITTLE architecture requires this, but more flexible power state management does too. Currently, processes are charged by the amount of time they spend executing in a CPU; that is clearly unfair to processes that are scheduled onto a slower processor. So charging will eventually have to change to a unit other than time; instructions executed, for example. The CPU scheduler will need to become more aware of the power management policies in force. Scheduling processes to enable the use of "turbo boost" mode (where a single CPU can be overclocked if all other CPUs are idle) remains an open problem. Thermal limits will throw more variables into the equation. And so on. 

It is also possible that the separation of CPU and power scheduling will not work out; as Morten put it: 

I'm aware that the scheduler and power scheduler decisions may be inextricably linked so we may decide to merge them. However, I think it is worth trying to keep the power scheduling decisions out of the scheduler until we have proven it infeasible. 

Even with these uncertainties, the "power scheduler" approach should prove to be a useful starting point; Morten and his colleagues plan to post a preliminary power scheduler implementation in the near future. At that point we may hear how Ingo feels about this design relative to the requirements he put forward; he (along with the other core scheduler developers) has been notably absent from the recent discussion. Regardless, it seems clear that the development community will be working on power-aware scheduling for quite some time.
