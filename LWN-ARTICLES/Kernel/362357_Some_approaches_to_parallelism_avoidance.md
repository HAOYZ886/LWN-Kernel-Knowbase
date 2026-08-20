---
title: Some approaches to parallelism avoidance
url: https://lwn.net/Articles/362357/
date: "November 17, 2009"
category: Scheduler
author: "By Jonathan Corbet November 17, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 17, 2009

What do you do if you have a group of processes, but only want one of them to run at any given time? This kind of workload is not that uncommon; it appears in user-space threading applications, asynchronous I/O applications, and in applications which have background processing tasks. Stijn Devriendt has such a problem; he recently [proposed](<https://lwn.net/Articles/362359/>) a solution in the form of a new system call: 

```
int sched_wait_block(pid_t pid, struct timespec *uts);
```

This call would put the process to sleep until the process indicated by `pid` blocked, at which point the calling process would go back onto the run queue. It would thus allow a sort of "only run me when process `pid` is sleeping" semantic. 

Ingo Molnar [responded](<https://lwn.net/Articles/362360/>) with a suggestion for a very different approach; to him, this problem is another nail for the "perf events" hammer. An interested process could sign up for "parallelism" events, then receive notifications when specific processes sleep or become runnable. He sees some real benefits from such a capability: 

This would make a very powerful task queueing framework. It basically allows a 'lazy' user-space scheduler, which only activates if the kernel scheduler has run out of work. 

Linus, though, had [a very different suggestion](<https://lwn.net/Articles/362361/>): rather than create this whole framework, just add a relatively stupid "only run one of this group of threads at a time" mode to the scheduler. This mode, which could be specified with a new `clone()` flag, seems like it could solve most of the problems in this area without adding a new set of complicated interfaces. 

As of this writing, only `sched_wait_block()` has an actual patch associated with it, and nobody has committed to writing any others. So the eventual outcome - if any - from this conversation is unclear at best, but it's an interesting exploration of approaches in any case.
