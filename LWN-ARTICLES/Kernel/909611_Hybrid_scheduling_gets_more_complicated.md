---
title: Hybrid scheduling gets more complicated
url: https://lwn.net/Articles/909611/
date: "September 30, 2022"
category: Scheduler
author: "By Jonathan Corbet September 30, 2022 LPC"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 30, 2022

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2022-Linux_Plumbers_Conference>)

Just over ten years ago, the Arm big.LITTLE architecture [posed a challenge](<https://lwn.net/Articles/481055/>) for the kernel's CPU scheduler: how should processes be assigned to CPUs when not all CPUs have the same capacity? The situation has not gotten simpler since then; new systems bring new quirks that must be kept in mind for optimal scheduling. At the [2022 Linux Plumbers Conference](<https://lpc.events>), Len Brown and Ricardo Neri talked about Intel's hybrid systems and the work that is being done to schedule properly on those systems. 

[![\[Len Brown\]](https://static.lwn.net/images/conf/2022/lpc/LenBrown-sm.png)](<https://lwn.net/Articles/909616/>) Brown started by describing Intel's hybrid CPUs which, he said, have a combination of "Pcores" and "Ecores". The Pcores have higher performance and also support simultaneous multi-threading (SMT). The Ecores, instead, are more focused on energy efficiency than performance; Ecores were once known as "Atom" CPUs. Both types of CPU implement the same instruction set, so a process can move freely between the two types. 

Kernel releases through 4.9 treated all CPUs on these systems as being equal; that meant that any given process would experience variable performance depending on where the scheduler placed it in the system. As of 4.10, Intel's ITMT (standing for "Intel Turbo Boost Max Technology") support caused the scheduler to prefer Pcores over Ecores, all else being equal. That had the effect of putting processes on the faster CPUs when possible, but it also would load all SMT sibling CPUs before falling back to the Ecores, which leads to worse performance overall. That has been fixed as of 5.16; an Ecore will now be preferred over an SMT CPU whose sibling is already busy. 

Pcores are faster; they run at a higher clock frequency, but are also able to get more work done with each clock cycle. As a result, clock frequencies alone are not sufficient to compare the capacity of two CPUs in a system. To address this problem, the hardware is able to provide both performance and efficiency scores for each CPU; these numbers can change at run time if conditions change, Brown said. 

The situation is actually a bit more complex than that, though. The performance difference between the CPU types depends on which instructions are being executed at any given time. Programs using the VNNI instructions (which are intended to accelerate machine-learning applications) may see much more advantage from running on a Pcore than those that are doing nothing special. There are four different classes of performance, dominated by instruction type, and the ratio of Pcore to Ecore performance is different for each. 

To schedule such a system optimally, the kernel should use the Pcores to run the processes that will benefit from them the most. Application developers cannot really be expected to know which of the four performance classes best describes their code, and the appropriate class may change over a program's execution in any case, but the CPU certainly knows which types of instructions are being executed at any given time. So each CPU exposes a register indicating which performance class best describes the currently running process. That allows the kernel to assign a class ID and use it in scheduling decisions. 

[![\[Ricardo Neri\]](https://static.lwn.net/images/conf/2022/lpc/RicardoNeri-sm.png)](<https://lwn.net/Articles/909617/>) Neri took over to describe the work that has been done to take advantage of this information. The class ID of each process is stored in its [`task_struct` structure](<https://elixir.bootlin.com/linux/v5.19.11/source/include/linux/sched.h#L726>). The first use of this information is in the idle load balancer, which is invoked when a CPU has run out of tasks to execute and looks to see if a task should be pulled from a more heavily loaded CPU elsewhere in the system. This code can look at the class ID of each candidate task to find the one that would benefit the most (or suffer the least) from being moved. This check works at both ends; a task that is making heavy use of instructions that are best run on its current CPU should not be moved if possible. 

An audience member asked whether the class ID of a running process can be adjusted from user space. Brown answered that this capability exists for debugging purposes, but that nobody had thought about making it available as a supported feature. 

Neri continued that the kernel's NUMA-balancing code can also look at the class IDs and exchange tasks between nodes if that would lead to better system performance. Something similar could also be done with busy load balancing, which tries to even out the load across a busy system. This idea made some developers nervous; it would be easy to break load balancing in ways that create performance regressions that don't come to light until long afterward. Neri emphasized that the class ID would only be used in load-balancing decisions if the existing heuristics led to a tie between two options. 

The final moments of the session were dedicated to the problem of scheduling on Intel's Alder Lake CPUs (which started shipping earlier this year). Specifically, the kernel's energy-aware scheduling heuristics don't work well on those CPUs. A number of features present there complicate the energy picture; these include SMT, Intel's "turbo boost" mode, and the CPU's internal power-management mechanisms. For many workloads, running on an ostensibly more power-hungry Pcore can be more efficient than using an Ecore. Time for discussion of the problem was lacking, though, and the session came to a close. 

[Thanks to LWN subscribers for supporting my travel to this event.]
