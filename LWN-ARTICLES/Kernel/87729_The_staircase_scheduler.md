---
title: The staircase scheduler
url: https://lwn.net/Articles/87729/
date: "June 2, 2004"
category: Interactivity; Scheduler; Staircase scheduler
author: 
---

As the 2.6.0 release approached, some developers worried that the CPU scheduler would be the downfall of this particular stable series. Complaints of poor interactive performance were common, NUMA systems were not supported well, and so on. Over time, most of these problems have been addressed; massive amounts of interactivity work and the domain scheduler have smoothed over most of the problems. Complaints about the scheduler have been relatively rare in recent times. 

One thing that does still bother some people, however, is the complexity of the current 2.6 scheduler. The interactivity work, in particular, added a great deal of very obscure code. The scheduler goes to great lengths to try to identify interactive tasks and to boost their priority accordingly. This process involves numerous strange computations involving a number of magic constants; it is difficult to understand, much less improve. 

Con Kolivas, who had his hand in much of the interactivity work, has just posted [a new version](<https://lwn.net/Articles/87244/>) of his "staircase scheduler" patch. This patch aims to greatly simplify the scheduler while simultaneously improving interactive response; it deletes 498 lines of code, while adding less than 200. Much of what is deleted is the "black magic" interactivity calculations; it is all replaced with a relatively simple, rank-based scheme. 

The staircase scheduler implements a single, ranked array of processes for each CPU. Initially, each process goes into the array at the rank determined by its base priority; the scheduler can then locate and run the highest-priority process in the usual way. So far, not much has changed. 

In the current scheduler, processes which use up their time slice get moved over to a separate "expired" array; there they languish until the rest of the processes in the mix have used up their time (or blocked) as well. The staircase scheduler does away with the expired array; instead, an expired process will be put back into the staircase, but at the next lower rank. It can, thus, continue to run, but at a lower priority. When it exhausts another time slice, it moves down again. And so on. The following little table shows how long the process spends at each priority level: 

| Priority rank  
---|---  
Iteration | Base | -1| -2| -3| -4| -5 | -6| -7| -8| -9| ...  
**1** | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1  
  
When a process falls off the bottom of the staircase, an interesting thing happens: it gets moved back up to one level below its previous maximum, and it gets two time slices at that level. Thereafter, it once again works its way down the steps to the bottom. The next time, it goes up to two steps below the maximum, for three time slices. The above table, with three iterations through the staircase, would look like this: 

| Priority rank  
---|---  
Iteration | Base | -1| -2| -3| -4| -5 | -6| -7| -8| -9| ...  
**1** | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1  
**2** | | 2| 1| 1| 1 | 1| 1| 1| 1| 1  
**3** | | | 3| 1| 1 | 1| 1| 1| 1|  1  
  
Each descent down the staircase thus involves the same number of time slices, but, each time, more slices are spent at the top priority level for that iteration. This algorithm helps maintain the relative priorities. A process at priority _n_ will, after falling off the staircase, find itself competing with all the processes at priority _n_ -1, but it will get a longer slice of time relative to those other processes, which have a lower base priority. 

If a process sleeps for a reasonable interval, it gets pushed back up the staircase. Thus interactive tasks, which normally sleep quite a bit, should stay near the top of the staircase and be responsive, while CPU hogs spend much of their time on the lower steps. 

The kernel community may not be up for another big scheduler change at this point in the stable series; many people would like to see 2.6 actually stabilize and 2.7 begin. This patch appears worthy of consideration, however, for its simplification of a complex part of the kernel if nothing else.
