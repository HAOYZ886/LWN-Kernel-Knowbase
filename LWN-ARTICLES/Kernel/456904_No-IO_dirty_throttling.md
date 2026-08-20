---
title: "No-I/O dirty throttling"
url: https://lwn.net/Articles/456904/
date: "August 31, 2011"
category: "Memory management-Writeback; Write throttling"
author: "By Jonathan Corbet August 31, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 31, 2011

"Writeback" is the process of writing dirty pages back to persistent storage, allowing those pages to be reclaimed for other uses. Making writeback work properly has been one of the more challenging problems faced by kernel developers in the last few years; systems can bog down completely (or even lock up) when writeback gets out of control. Various approaches to improving the situation have been discussed; one of those is Fengguang Wu's I/O-less throttling patch set. These changes have been circulating for some time; they are seen as having potential - if only others could actually understand them. Your editor doesn't understand them either, but that has never stopped him before. 

One aspect to getting a handle on writeback, clearly, is slowing down processes that are creating more dirty pages than the system can handle. In current kernels, that is done through a call to `balance_dirty_pages()`, which sets the offending process to work writing pages back to disk. This "direct reclaim" has the effect of cleaning some pages; it also keeps the process from dirtying more pages while the writeback is happening. Unfortunately, direct reclaim also tends to create terrible I/O patterns, reducing the bandwidth of data going to disk and making the problem worse than it was before. Getting rid of direct reclaim has been on the "to do" list for a while, but it needs to be replaced by another means for throttling producers of dirty pages. 

That is where Fengguang's [patch set](<https://lwn.net/Articles/456503/>) comes in. He is attempting to create a control loop capable of determining how many pages each process should be allowed to dirty at any given time. Processes exceeding their limit are simply put to sleep for a while to allow the writeback system to catch up with them. The concept is simple enough, but the implementation is less so. Throttling is easy; performing throttling in a way that keeps the number of dirty pages within reasonable bounds and maximizes backing store utilization while not imposing unreasonable latencies on processes is a bit more difficult. 

If all pages in the system are dirty, the system is probably dead, so that is a good situation to avoid. Zero dirty pages is almost as bad; performance in that situation will be exceedingly poor. The virtual memory subsystem thus aims for a spot in the middle where the ratio of dirty to clean pages is deemed to be optimal; that "setpoint" varies, but comes down to tunable parameters in the end. Current code sets a simple threshold, with throttling happening when the number of dirty pages exceeds that threshold; Fengguang is trying to do something more subtle. 

Since developers have complained that his work is hard to understand, Fengguang has filled out the code with lots of documentation and diagrams. This is how he depicts the goal of the patch set: 

```
^ task rate limit
      |
      |            *
      |             *
      |              *
      |[free run]      *      [smooth throttled]
      |                  *
      |                     *
      |                         *
      ..bdi->dirty_ratelimit..........*
      |                               .     *
      |                               .          *
      |                               .              *
      |                               .                 *
      |                               .                    *
      +-------------------------------.-----------------------*------------>
                              setpoint^                  limit^  dirty pages
```

The goal of the system is to keep the number of dirty pages at the setpoint; if things get out of line, increasing amounts of force will be applied to bring things back to where they should be. So the first order of business is to figure out the current status; that is done in two steps. The first is to look at the global situation: how many dirty pages are there in the system relative to the setpoint and to the hard limit that we never want to exceed? Using a cubic polynomial function (see [the code](<http://permalink.gmane.org/gmane.linux.kernel.mm/67529>) for the grungy details), Fengguang calculates a global "pos_ratio" to describe how strongly the system needs to adjust the number of dirty pages. 

This ratio cannot really be calculated, though, without taking the backing device (BDI) into account. A process may be dirtying pages stored on a given BDI, and the system may have a surfeit of dirty pages at the moment, but the wisdom of throttling that process depends also on how many dirty pages exist for that BDI. If a given BDI is swamped with dirty pages, it may make sense to throttle a dirtying process even if the system as a whole is doing OK. On the other hand, a BDI with few dirty pages can clear its backlog quickly, so it can probably afford to have a few more, even if the system is somewhat more dirty than one might like. So the patch set tweaks the calculated pos_ratio for a specific BDI using a complicated formula looking at how far that specific BDI is from its own setpoint and its observed bandwidth. The end result is a modified pos_ratio describing whether the system should be dirtying more or fewer pages backed by the given BDI, and by how much. 

In an ideal world, throttling would match the rate at which pages are being dirtied to the rate that each device can write those pages back; a process dirtying pages backed by a fast SSD would be able to dirty more pages more quickly than a process writing to pages backed by a cheap thumb drive. The idea is simple: if N processes are dirtying pages on a BDI with a given bandwidth, each process should be throttled to the extent that it dirties 1/N of that bandwidth. The problem is that processes do not register with the kernel and declare that they intend to dirty lots of pages on a given BDI, so the kernel does not really know the value of N. That is handled by carrying a running estimate of N. An initial per-task bandwidth limit is established; after a period of time, the kernel looks at the number of pages actually dirtied for a given BDI and divides it by that bandwidth limit to come up with the number of active processes. From that estimate, a new rate limit can be applied; this calculation is repeated over time. 

That rate limit is fine if the system wants to keep the number of dirty pages on that BDI at its current level. If the number of dirty pages (for the BDI or for the system as a whole) is out of line, though, the per-BDI rate limit will be tweaked accordingly. That is done through a simple multiplication by the pos_ratio calculated above. So if the number of dirty pages is low, the applied rate limit will be a bit higher than what the BDI can handle; if there are too many dirty pages, the per-BDI limit will be lower. There is some additional logic to keep the per-BDI limit from changing too quickly. 

Once all that machinery is in place, fixing up `balance_dirty_pages()` is mostly a matter of deleting the old direct reclaim code. If neither the global nor the per-BDI dirty limits have been exceeded, there is nothing to be done. Otherwise the code calculates a pause time based on the current rate limit, the pos_ratio, and number of pages recently dirtied by the current task and sleeps for that long. The maximum sleep time is currently set to 200ms. A final tweak tries to account for "think time" to even out the pauses seen by any given process. The end result is said to be a system which operates much more smoothly when lots of pages are being dirtied. 

Fengguang has been working on these patches for some time and would doubtless like to see them merged. That may yet happen, but adding core memory management code is never an easy thing to do, even when others can easily understand the work. Introducing regressions in obscure workloads is just too easy to do. That suggests that, among other things, a lot of testing will be required before confidence in these changes will be up to the required level. But, with any luck, this work will eventually result in better-performing systems for us all.
