---
title: Issues with epoll()
url: https://lwn.net/Articles/637435/
date: "March 23, 2015"
category: Epoll
author: "By Jake Edge March 23, 2015 LSFMM 2015"
---

By **Jake Edge**  
March 23, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

In a filesystem session at the 2015 LSFMM Summit, Jason Baron led a discussion about the `epoll()` system call. He and others have [observed some performance problems with `epoll()`](<https://lwn.net/Articles/633422/>), especially for large sets of monitored file descriptors. There are two problems that Baron is trying to address: the "thundering herd" problem on wakeups and the use of global locks when manipulating the `epoll()` sets. He has posted patches for both, but they haven't really been commented on, he said. He also noted that Fam Zheng has posted some patches that add new system calls for `epoll()`. 

[ ![\[Jason Baron\]](https://static.lwn.net/images/2015/lsf-baron-sm.jpg) ](<https://lwn.net/Articles/637439/>)

The thundering herd problem occurs when there are multiple threads that share a wakeup source in their `epoll()` sets. When that file descriptor becomes ready, _all_ of the threads waiting wake up, even though only one of them is needed to service the event. One solution that had been suggested was to have a single `epoll()` queue, with all events being taken off that single queue. But that is not optimal for what he is trying to do, he said. 

His patches simply wakeup the first idle thread that is waiting, then round-robin through the threads on subsequent wakeups. Some suggested using CPU affinity to wake up the thread on the CPU where the interrupt has come in. But `epoll()` doesn't currently have access to that kind of information. Baron has "heard vaguely" that some people are doing this, but he hasn't seen any patches. He would like to explore the idea further. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

His initial proposal was to simply wake up one thread waiting on the `epoll()` set, but there was concern that might break programs that were expecting the current behavior. The wait queue used is associated with a file descriptor, so it is local to the process (and its threads), rather than global. A flag passed to `epoll()` could change the behavior for a program without affecting other programs that might also be waiting. 

Another option that he has tried is to change the wakeup behavior in the scheduler, though he was worried that the scheduler developers would be unhappy with a change like that. When he posted it, though, there was no feedback of that sort. Still, avoiding changes to the wakeup code is desirable. 

But `epoll()` has the ability to nest the file descriptors it is monitoring. That means a set of file descriptors can be constructed that contains descriptors returned from other `epoll_create1()` calls. In the past, loops could be created that way, though that has been fixed. One could use the nesting capability, coupled with a new flag to `epoll_create1()` to add the round-robin feature, but restrict the changes to the `epoll()` code instead of changing the wakeup code. 

Jeff Layton asked if there would be two flags, one to request the CPU affinity mode and one for the round-robin behavior. But Baron did not think both would be needed. The CPU affinity mode could simply fall back to round-robin behavior if the interrupt did not come in on a CPU that was running a thread waiting on the event. 

He moved on to locking, which has shown up in some profiles of `epoll()` performance. Akamai (where Baron works) has not necessarily run into it, but people don't like global locks, in general, he said. Part of the problem is that the kernel does not know when the sets have file descriptors in common, so it locks everything when manipulating them. 

The idea is to break up the locks in the classic way, he said, so that operations are serialized only for sets with common file descriptors. He posted patches a few months ago, but they added three pointer fields to `struct file`, which was not something other developers were happy with. He plans to switch to only adding a single pointer that points to a structure to hold anything that `epoll()` needs. It would be allocated when the `epoll()` file descriptor is created. 

In addition, his patches eliminate the runtime checking for loops and too deep of nesting in the file descriptor sets. Right now those checks are done when calling `epoll_wait()`, but his patches do that checking when file descriptors are added to the set in `epoll_ctl()`. 

Layton asked if all of this work meant that Baron was volunteering to be the `epoll()` maintainer. Baron was non-committal, but Chris Mason suggested (with a chuckle) that if these patches were accepted, that would more or less happen by default. 

Mason said that Facebook is hitting some of these problems, as is Google. Someone said that GlusterFS is hitting them too. Baron said that Akamai would be using his patches in production, so they should get lots of testing. 

There are other `epoll()` patches out there, including those for new system calls from Zheng. Others include a patch that would add a lockless way to enqueue and dequeue events and one that would optimistically wait (briefly) in the kernel for another event rather than immediately go to sleep. The person working on the latter patches, which were targeted at networking, is now working on other things, Matthew Wilcox said, so they could be taken over by someone else if that was of interest. 

It would seem that scalability problems with `epoll()` are cropping up in a number of places, so some fixes are needed. Baron's patches are not running into much in the way of opposition, at least from the assembled filesystem developers, which means they may make their way into the mainline before long. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
