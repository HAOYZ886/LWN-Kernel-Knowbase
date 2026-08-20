---
title: Measuring memory fragmentation
url: https://lwn.net/Articles/974943/
date: "May 28, 2024"
category: "Memory management-Huge pages"
author: "By Jonathan Corbet May 28, 2024 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 28, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

In the final session in the memory-management track of the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), the exhausted group of developers looked one more time at the use of huge pages and the associated problem of memory fragmentation. At its worst, this problem can make huge pages harder (and more expensive) to allocate. Luis Chamberlain, who ran the session, felt that people were worried about this problem, but that there was little data on how severe it truly is. 

Transparent huge pages, he said, never reached wide adoption, partly as the result of fragmentation fears. But now, the kernel supports large folios, and transparent huge pages are "the stone age". Large folios are being used in a number of places, and multi-size transparent huge pages (mTHPs) are on the rise as well — and "the world hasn't ended". Still, worries abound, so he wondered how the fragmentation problem could actually be measured. 

[![\[Luis
Chamberlain\]](https://static.lwn.net/images/conf/2024/lsfmm/LuisChamberlain-sm.png)](<https://lwn.net/Articles/974948/>) The discussion immediately wandered. David Hildenbrand said that there are people who have been looking into allocation failures and running into the fragmentation problem. SeongJae Park pointed out that, long ago, Mel Gorman had [proposed a fragmentation index](<https://lore.kernel.org/lkml/1268412087-13536-7-git-send-email-mel@csn.ul.ie/>) that was since merged as a debugfs feature, and that some of Gorman's team are using it. Michal Hocko said that it is a question of proactive or reactive responses; at what level should people care about fragmentation? Hildenbrand said that, currently, most allocations will fall back to a base page if larger pages are not available; in the future, if users need the larger allocations, that fallback will no longer be an option. There will be a need to measure the availability of specific allocation sizes to understand the fragmentation problem, he said. 

In response to a question from Hocko on the objective for this measurement, Chamberlain said that he wanted to know if [the introduction of large block sizes](<https://lwn.net/Articles/974219/>) was making fragmentation worse. And, if the fragmentation problem is eventually solved, how do we measure it? Hocko suggested relying on the [pressure-stall information](<https://lwn.net/Articles/759781/>) provided by the kernel; it is measuring the amount of work that is needed to successfully allocate memory. But he conceded that it is "a ballpark measure" of the problem. 

Yu Zhao said that kernel developers cannot improve what they cannot measure; Paul McKenney answered that they can always improve things accidentally. That led Zhao to rephrase his point: fragmentation, he said, is a form of entropy, which is typically measured by temperature. But fragmentation is a two-dimensional problem that cannot be described by a single number. Any proper description of fragmentation, he said, will need to be multidimensional. Jan Kara said that a useful measurement would be the amount of effort that is going into memory compaction, but Zhao repeated that a single number will never suffice. 

John Hubbard disagreed, saying that it should be possible to come up with a single number quantifying fragmentation; Zhao asked how that number would be interpreted. Hocko said that there is an important detail that would be lost in a single-number measurement: the view of fragmentation depends on a specific allocation request. Movable allocations are different from `GFP_KERNEL` allocations, for example. He said that, in any case, a precise number is not needed; he repeated that the pressure-stall information shows how much nonproductive time is being put into memory allocations, and thus provides a good measurement of how well things are going. 

As the session wound down, Chamberlain tried to summarize the results, which he described as being "not a strong argument" for any given measure. Zhao raised a specific scenario: an Android system running three apps, one in the foreground and two in the background. There is a single number describing fragmentation, and allocations are failing; what should be done? Possible responses include memory compaction, reclaim, or summoning the out-of-memory (OOM) killer; how will this number help to make this decision? Chamberlain said that he is focused on the measurement, not the reactions at this point. Zhao went on for a while about how multi-dimensional measurements are needed to address this problem before Hocko said that the topic could be discussed forever without making much progress; the session then came to a close.
