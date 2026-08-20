---
title: "Per-file OOM badness"
url: https://lwn.net/Articles/896738/
date: "June 2, 2022"
category: "Memory management-Out-of-memory handling"
author: "By Jonathan Corbet June 2, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 2, 2022

The kernel tries hard to keep memory available for its present and future needs. Should that effort fail, though, the tool of last resort is the dreaded out-of-memory (OOM) killer, which is tasked with killing processes on the system to free their memory and alleviate the problem. The results of invoking the OOM killer are never going to be good, but they can be distinctly worse if the wrong processes are chosen for an untimely end. As one might expect, the effort to properly choose the right processes is an ongoing effort. Most recently, Christian König has proposed [a new mechanism](<https://lwn.net/ml/linux-kernel/20220531100007.174649-1-christian.koenig@amd.com/>) to address a blind spot in the OOM killer's deliberations. 

When the system runs out of memory, the OOM killer's job is to try to resolve the problem while causing the least possible amount of collateral damage; a number of heuristics have been applied to the victim-choosing logic toward that end. One obvious rule is that it is generally better to kill fewer processes than many, and the way to do that is to select the processes that are currently consuming the most memory. Often, a single out-of-control process is responsible for the problem in the first place; if that process can be identified and killed, the system can get back into a more stable condition. 

The OOM killer, thus, scans through the set of running processes to find the most interesting target. At the core of this calculation is a function called [`oom_badness()`](<https://elixir.bootlin.com/linux/v5.18.1/source/mm/oom_kill.c#L192>), which sums up the amount of memory (and swap space) being used by a candidate process. That sum is then adjusted by the process's [`oom_score_adj`](<https://docs.kernel.org/filesystems/proc.html#chapter-3-per-process-parameters>) value (which is a knob that an administrator can tweak to direct the OOM-killer's attention toward or away from specific processes) before being returned as the process's score. The process with the highest score as determined by this function will be the first on the chopping block. Any process's score can be seen at any time by reading its `oom_score` value from its `/proc` entry. 

One problem with this algorithm, as identified by König, is that `oom_badness()` does not take into account all of the memory used by a process. Specifically, memory associated with files is not counted; consider, for example, any extra memory that a device driver must allocate when a device special file is opened and operated upon. For some workloads, this memory can be significant, with the result that the processes accounting for the most memory use might not look like attractive OOM-killer targets. 

As a simple example, he said in the patch-series cover letter, a malicious process can call [`memfd_create()`](<https://man7.org/linux/man-pages/man2/memfd_create.2.html>), then just write indefinitely to the resulting memfd; the memory consumed by the memfd will not be seen as belonging to the offending process so, when the memfd ends up consuming all of the available memory, the OOM killer will pass over that process. This sequence ""can bring down any standard desktop system within seconds"". Another problem area, he said, is graphics applications that allocate significant amounts of memory within the kernel for graphical resources. 

The solution is to give the OOM killer visibility into the memory resources that are consumed in this way. That, in turn, involves adding yet another member to the ever-growing [`file_operations` structure](<https://elixir.bootlin.com/linux/v5.18.1/source/include/linux/fs.h#L1957>): 

```
long (*oom_badness)(struct file *file);
```

Documentation is lacking, but the intent seems to be that this function, if it exists, should return the amount of extra memory attached to the given `file`, in pages. This function will be called from within the global `oom_badness()` function to take that extra memory usage into account; if the `file` involved is shared between processes, the memory usage will be divided equally among those processes. 

Implementations of the new function have been added to the shared-memory filesystem code, the DMA-buf subsystem, and to most graphics drivers. With this mechanism in place, the system has a better idea of where the OOM killer's wrath should be directed to maximize the chances of freeing up significant amounts of memory and bringing the system back to a stable state. 

Of course, the hazards of any new heuristic can be seen in this claim in the cover letter: accounting for this memory, König says, provides ""a quite a bit better system stability in OOM situations, especially while running games"". Accounting for memory used by graphics drivers is likely to point the finger at graphics-intensive applications — games, for example — as the source of an out-of-memory problem. Having the OOM killer take its vengeance on that game may restore the system, but the user, whose nearly complete quest would be abruptly terminated thereby, might be forgiven for thinking that the situation was better before. 

In other words, there is still no truly good solution to the OOM problem other than not getting into that situation in the first place. After all, the OOM killer is still, as Andries Brouwer [suggested in 2004](<https://lwn.net/Articles/104185/>), like choosing passengers to toss out of a crashing aircraft. When the system runs out of memory anyway, though, it is important to free memory quickly, and that is most likely to happen if the OOM killer has an accurate picture of which processes are using the most memory. Properly accounting for memory attached to files seems like a useful step in that direction.
