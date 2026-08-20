---
title: "KS2007: Memory management"
url: https://lwn.net/Articles/249054/
date: "September 10, 2007"
category: "Memory management-Conference sessions"
author: "By Jonathan Corbet September 10, 2007 LWN.net Kernel Summit 2007 coverage"
---

By **Jonathan Corbet**  
September 10, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

The kernel summit session on memory management was led by Mel Gorman and Peter Zijlstra. While the VM hackers have a lot going on, this session was dominated by three topics: large page support, test cases, and memory pressure notification. 

There continues to be pressure for improved large-page support on Linux systems. For almost any architecture, proper use of large pages can help to relieve pressure on the translation lookaside buffer (TLB), with a corresponding increase in performance. Some architectures (SuperH, for example) have very small TLBs and, thus, a large motivation to use large pages whenever possible. This would be easier to do if Linux could support more than one size of large pages. Some processors have several different size options, some up to 1GB. 

Large pages are currently made available via hugetlbfs, an interface which application developers have, in general, not yet learned to love. Hugetlbfs currently only provides a single size of large pages, so providing multiple page sizes will require an extension to this virtual filesystem. Initially, an extension might take a relatively rudimentary form, such as a mount-time page size option. Multiple sizes could then be accommodated by mounting hugetlbfs multiple times. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There are challenges involved in supporting some of these page sizes, though. 1GB pages are currently larger than `MAX_ORDER`, the largest chunk of contiguous (small) pages that the kernel tracks. Increasing `MAX_ORDER` is a bit more work than just changing a definition somewhere. Different sizes of pages also have to be established at different levels in the page table hierarchy, something which is not currently well supported by the kernel's page table API. Linus cut short discussion on API issues, though, warning against any attempts to generalize the generic API for all of the large page issues. So much of this problem is so incredibly architecture-specific that trying to solve it in generic code is likely to lead to bigger messes than it solves. So much of the work for large-page support will probably have to be done in architecture-specific code. 

Mel spent much of the session trying to get the larger group to agree on what a proper test case for memory management patches is. Or, even if they wouldn't agree, to just get some suggestions for what could be a good test case. It would appear that he has grown just a little bit weary of being told that his patches need to be benchmarked on a real test case before they can be considered for inclusion. He seems willing to do that benchmarking, but, so far, nobody has stepped forward and told him what kind of "real workload" they are expecting him to use. 

He got little satisfaction at the summit. The problem is that some kinds of workloads are relatively easy to benchmark, but other kinds of parameters ("interactivity") are hard to measure. So, even if somebody could put together in implementation of (say) swap prefetch, there is no real way to prove that it is actually useful. And, in the absence of such proof, memory management patches are notoriously hard to merge. There were not a whole lot of ideas for improving the situation. Your editor can say, though, that he will go out of his way not to be the next reviewer to ask Mel about which real workloads he has tested a patch on. 

The final topic was working out a way to let applications help when the system is under memory pressure. Web browsers, for example, often maintain large in-memory caches which can be dropped if the system finds itself running out of memory - but that will only happen if the browser knows about the problem. There are other applications in a similar situation; GNOME and KDE applications, for example, tend to carry a certain amount of cached data which can be done without if the need arises. 

The problem is figuring out how to tell the application that the time has come to free up some memory. Sending a signal might be an obvious way to send a notification, but nobody really wants to extend the signal interface. Responses to memory pressure notifications must often be done in libraries, and working with signals in library code is especially problematic. In the absence of signals, there will have to be a way for applications to somehow ask about memory pressure. 

After a brief digression into the rarefied, philosophical question of just what is memory pressure in the first place, the discussion wandered into a different approach to the problem. Perhaps an application could make a system call to indicate that it does not currently need a specific range of memory, but, if the system doesn't mind, keeping it around might just be useful. If, at some future point, the application wants something that it had cached, it makes another call to query whether the given range of memory is still there. This would give the kernel a list of pages it could dump if it finds itself in a tight spot, but still keeps the data around if there is not a pressing need for that memory. 

Linus cautioned that these system calls might seem like a nice idea, but that nobody would ever use them. In general, he says, Linux-specific extensions tend not be used. Developers do not want to maintain any more system-specific code than they really have to. Some people thought that there might be motivation for a few library developers to use these calls, though. But until such a time as a patch implementing them actually exists, this discussion will probably not go a whole lot further.
