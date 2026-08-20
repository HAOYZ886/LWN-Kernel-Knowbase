---
title: "Negative dentries, 20 years later"
url: https://lwn.net/Articles/890025/
date: "April 11, 2022"
category: Dentry cache
author: "By Jonathan Corbet April 11, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 11, 2022

Filesystems and the virtual filesystem layer are in the business of managing files that actually exist, but the Linux "dentry cache", which remembers the results of file-name lookups, also keeps track of files that _don't_ exist. This cache of "negative dentries" plays an important role in the overall performance of the system but, if it is allowed to grow too large, its role can become negative in its own right. As the 2022 [Linux Storage, Filesystem, and Memory-Management Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM) approaches, the subject of negative dentries has come up yet again; whether one can be positive about the prospects for a resolution this time around remains unclear. 

The kernel's dentry cache saves the results of looking up a file in a filesystem. Should the need arise to look up the same file again, the cached result can be used, avoiding a trip through the underlying filesystem and accesses to the storage device. Repeated file-name lookups are common — consider `/usr/bin/bash` or `~/.nethackrc` — so this is an important optimization to make. 

The importance of remembering failed lookups in negative dentries may be less obvious at the outset. As it happens, repeated attempts to look up a nonexistent file are also common; an example would be the shell's process of working through the search path every time a user types "`vi`" (Emacs users start the editor once and never leave its cozy confines thereafter, so they don't benefit in the same way). Even more common are failed lookups created by the program loader searching for shared libraries or a compiler looking for include files. One is often advised to "fail fast" in this society; when it comes to lookups of files that don't exist, that can indeed be good advice. 

So negative dentries are a good thing but, as we all know, it is possible to have too much of a good thing. While normal dentries are limited by the number of files that actually exist, there are few limits to the number of nonexistent files. As a result, it is easy for a malicious (or simply unaware) application to create negative dentries in huge numbers. If memory is tight, the memory-management subsystem will eventually work to push some of these negative dentries out. In the absence of memory pressure, though, negative dentries can accumulate indefinitely, leaving a large mess to clean up when memory does inevitably run out. 

Some kernel problems are resolved quickly; others take a little longer. LWN briefly [reported on](<https://lwn.net/Articles/1511/>) a complaint about the memory consumption of negative dentries back in 2002, nearly exactly 20 years ago. A more recent attempt to solve the problem was [covered here](<https://lwn.net/Articles/814535/>) in early 2020. While numerous developers have taken a stab at the negative-dentry problem over time, the core problem remains. Those dentries still take up valuable memory, and they can create other problems (such as [soft lockups](<https://lwn.net/ml/linux-fsdevel/20220209231406.187668-1-stephen.s.brennan@oracle.com/>)) as well. 

#### A new discussion

In mid-March, Matthew Wilcox [suggested](<https://lwn.net/ml/linux-mm/YjDvRPuxPN0GsxLB@casper.infradead.org/>) that the negative-dentry problem might make a good LSFMM topic: ""maybe some focused brainstorming on the problem would lead to something that actually works"". Often, simply proposing a topic like this can elicit the sort of brainstorming needed to work toward a solution. That didn't happen this time, but it did lead to the posting of [a patch set](<https://lwn.net/ml/linux-kernel/20220331190827.48241-1-stephen.s.brennan@oracle.com/>) by Stephen Brennan showing a new approach to the problem. 

One of the difficulties posed by the negative-dentry problem is that it can be hard to know when the time has come to start throwing them away. The sizes of systems and workloads vary hugely, so any sort of simple limit is likely to cause performance regressions somewhere. Providing a knob for the system administrator to tune the limit can be tempting, but that just pushes the problem onto the users, and it is generally felt that the kernel should be able to figure things out by itself. But, as Brennan noted, that is not easy: 

> It's hard to look at a hash bucket or LRU list and design a heuristic for an acceptable amount of negative dentries: it won't scale from small to large systems well. But setting up heuristics on a per-directory basis will scale better, and it's easier to reason about. 

The specific heuristic proposed by the patch is that the negative dentries for any given directory should not outnumber the positive dentries by more than a factor of five. If there are 20 positive dentries in the cache for a directory, there can be no more than 100 negative dentries. It is a nice idea, with only one small problem: the kernel doesn't keep counts of the number of dentries (or their types) associated with each directory. 

To get around that, Brennan added code that maintains a "cursor" in the list of dentries associated with each directory. Whenever a dentry operation (creation or deletion) happens, that code will advance the cursor through the next six dentries in the list; if it does not encounter at least one positive dentry, it assumes that the limit has been exceeded and cleans up some negative dentries. Attaching this work to the dentry operations themselves means that the penalty will be paid by processes that are responsible for the creation of a lot of dentries, which seems correct. 

The problem with this approach is, of course, that there is nothing that forces dentries to be added to a directory's list in any particular order. Depending on the order in which dentries are created, this algorithm could come to an incorrect conclusion regarding the real ratio of positive to negative dentries and do the wrong thing. Brennan acknowledged this problem (""This workload-dependence is bad, full stop""), but has not yet come up with a better idea. As things stand, this algorithm seems certain to lead to pathological cases; that may prevent the acceptance of this patch set even in the absence of other concerns. 

#### The bigger problem

Back in the general discussion, though, Dave Chinner [argued](<https://lwn.net/ml/linux-mm/20220316025223.GR661808@dread.disaster.area/>) that the focus on negative dentries was addressing a symptom of the problem and missing the bigger issue. The real problem, he said, is that memory pressure is the only mechanism the kernel has for controlling the size of the many caches it maintains: 

> Yup, the underlying issue here is that memory reclaim does nothing to manage long term build-up of single use cached objects when *there is no memory pressure*. There's [plenty] of idle time and spare resources to manage caches sanely, but we don't. e.g. there is no periodic rotation of caches that could lead to detection and reclaim of single use objects (say over a period of minutes) and hence prevent them from filling up all of memory unnecessarily and creating transient memory reclaim and allocation latency spikes when memory finally fills up. 

Rather than worry about the dentry cache, he said, developers should come up with a mechanism that can manage the size of all in-kernel caches. Wilcox [agreed in principle](<https://lwn.net/ml/linux-mm/YjnmcaHhE1F2oTcH@casper.infradead.org/>), but cautioned against making the problem so broad that it becomes intractable. Chinner [doubled down](<https://lwn.net/ml/linux-mm/20220322222114.GE1609613@dread.disaster.area/>), though, saying that multiple kernel caches have the same problem, and that a solution for one, based on some sort of periodic scanning to age items out of the cache, would be instantly applicable to all of them. 

This discussion has mostly wound down without any suggestion that anybody is setting out to create the more general cache-aging mechanism that Chinner would like to see. The problem remains, though, and seems unlikely to go away by itself. So the chances of this discussion showing up in an LSFMM slot seem fairly high. Perhaps an in-person discussion — the first in the memory-management and filesystem communites in three years — will lead to some sort of consensus on a solution, preferably one that will be implemented before another 20 years pass.
