---
title: "KS2012: memcg/mm: Hierarchical reclaim for memory cgroups"
url: https://lwn.net/Articles/516535/
date: "September 17, 2012"
category: "Memory management-Control groups"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

As background to the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>) session on hierarchical reclaim, it's useful to note that somewhat like hard and soft quotas for disk usage, memory cgroups can have soft and hard limits on memory usage. Memory cgroups can grow above a soft limit in the absence of global memory pressure. In the event of global memory pressure, the intention is that memory cgroups that are above their soft limits should be the first to shrink. In any case, memory cgroups can never exceed their hard limit. More detail on soft limits can be found in the kernel source file `[Documentation/cgroups/memory.txt](<https://lwn.net/Articles/516022/>)`. 

Ying Han talked in detail about [soft-limit reclaim](<https://lwn.net/Articles/516438>) for hierarchical memory cgroups. The scenario of interest here concerns soft-limited cgroups that have child cgroups. The question is how to apply page reclaim pressure on those children if their parent is over its soft limit. 

Ying began with a basic example of a cgroup tree that looks like this: 

```
root
    	/ | \
           A  B  C
```

(Each node in these diagrams is a cgroup; in this diagram, `root` is the parent of three child cgroups.) 

In the event of global memory pressure, cgroups that are above their soft limits will shrink; this is relatively straightforward. The situation becomes more complex when there are child cgroups of these cgroups: 

```
root
    	/ | \
           A  B  C
          / \
         A1 A2
```

If `A1` and `A2` are below their limits and `A` is above its limit, then a whole subtree needs to shrink. This gets even worse if the soft limit of `A` is less than the sum of all the soft limits of its children. (It would be unusual to have a configuration where the sum of the children's soft limits exceeded that of the parent, but it is not one that is explicitly forbidden.) There is no real agreement on the semantics of how the hierarchy should be walked and pages reclaimed. The basic suggestion is: 

  1. If any of the child cgroups are above the soft limit, then reclaim from them, and do not follow down the hierarchy. 

  2. If all the child cgroups are below their limit, then reclaim from some or all of them proportionally. 

A second suggestion was to declare the configuration invalid and forbid it from happening, but this was not a popular choice because there are some use cases where such a configuration is desired. 

The discussion focused on the semantics of how the tree should be walked, and in what order cgroups should be reclaimed from; Rik van Riel discussed a proposal on how to calculate the number of pages to reclaim from each group. There was very little consensus on how hierarchical reclaim should be handled, but it is likely that multiple tree walks will be necessary to cover all cases. The problem is complex, but at the very least the discussion means that it will be easier to understand the motivation behind related patches posted in the future. 

[Next: Reclaiming mapped pages](<https://lwn.net/Articles/516536/>)
