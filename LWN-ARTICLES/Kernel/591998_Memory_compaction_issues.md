---
title: Memory compaction issues
url: https://lwn.net/Articles/591998/
date: "March 26, 2014"
category: "Memory management-Large allocations"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

Memory compaction is the process of relocating active pages in memory in order to create larger, physically contiguous regions — memory defragmentation, in other words. It is useful in a number of ways, not the least of which is making huge pages available. But compaction apparently has some problems of its own; Vlastimil Babka led a brief session in the 2014 Linux Storage, Filesystem, and Memory Management Summit to explore the issue. 

After Vlastimil gave a quick overview of how compaction works (also described in [this article](<https://lwn.net/Articles/368869/>)) and described problems related to compaction overhead, Rik van Riel made the claim that there are two core issues to be looked at in this area: (1) can the compaction code be made to be faster, and (2) when compaction appears to be too expensive, should it just be skipped? 

It seems that a number of compaction bugs have been fixed over the years, but some clearly remain. How, it was asked, can they be made easier to find? Writing test programs that reveal compaction problems tends to be hard; these problems arise out of specific workloads that exercise the system in certain ways. There does not appear to be any easy way to abstract the problematic access patterns out of the workloads into separate test programs. 

What that means is that the memory management developers don't really have a good understanding of why compaction problems are happening. Some workloads obviously create situations where compaction gets expensive, but how that happens is obscure. So there is clearly a need to gain a better understanding of how the problems come about. One step in that direction might be to add a new counter that is incremented anytime the kernel detects that it has spent a significant amount of time in the compaction code. If that counter starts to increase, that will be a signal that bugs in the compaction code are being tickled. Then, perhaps, it will be possible to try to figure out where those bugs are. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
