---
title: A 2023 DAMON update
url: https://lwn.net/Articles/931769/
date: "May 16, 2023"
category: "Memory management-DAMON"
author: "By Jonathan Corbet May 16, 2023 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 16, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

[DAMON](<https://lwn.net/Articles/812707/>) is a framework that allows user space to influence and control the kernel's memory-management operations. It first entered the kernel with the 5.15 release, and has been gaining capabilities ever since. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), DAMON author Seongjae Park provided an overview of the current status of DAMON development and where it can be expected to go in the near future. 

At its core, Park began, DAMON provides access-frequency monitoring of memory in both the physical and virtual address spaces. It includes a number of methods to reduce the overhead of that monitoring. [DAMOS](<https://lwn.net/Articles/863753/>) is an add-on that allows this access information to be turned into memory-management decisions. It works by identifying regions of memory that match a specified access pattern, then applying policies to that region. DAMOS includes a quota mechanism that limits its own CPU overhead and can prioritize certain types of regions. Park has added various APIs over time, enhancing the user-space API and adding actions for control of memory-reclaim operations. 

[![\[SeongJae Park\]](https://static.lwn.net/images/conf/2023/lsfmm/SeongJaePark-sm.png)](<https://lwn.net/Articles/931771/>) A number of features have been added in the last year. DAMON gained online tuning in 5.19, making it easier to tweak its operating parameters in a running system. [LRU-list manipulation](<https://lwn.net/Articles/905370/>) arrived in 5.20; it works by detecting hot regions of memory and marking the pages found therein as active. This mechanism, Park said, can reduce memory-pressure stalls by 20% when properly fine-tuned. "DAMOS tried regions" showed up in 6.2; it allows for inspection of memory regions in detail and supersedes an older API based on tracepoints. Tried regions are mostly useful for debugging DAMOS policies, he said. DAMOS filters, merged for 6.3, allow policies to limit actions to given types of pages — only anonymous pages, for example, or those belonging to a specific control group. 

Looking forward, Park is planning to add a feature for the monitoring of read-only and write-only access patterns. This could be used, for example, to identify processes that should be migrated to different types of backing store or those that would benefit from [kernel samepage merging](<https://www.kernel.org/doc/html/latest/admin-guide/mm/ksm.html>). He has posted a patch set implementing this feature in the past, but it was blocked due to unrelated memory-management problems; perhaps that situation has been resolved by now, he said. David Hildenbrand pointed out that write-only tracking would not be possible with pages that are pinned in memory, so that data could be incomplete; Park answered that, since DAMON uses sampling to limit overhead, it can't track every single page in any case and pinning doesn't make things any worse. 

Park's top priority, he said, might be "feedback-based quota auto tuning". Getting the most out of DAMON requires fine-tuning it to just the right level of aggressiveness; that can be tricky, and the optimal tuning may change over time. A poorly tuned implementation may impose too much overhead on the system, or it may not produce the desired results. To make life easier for administrators, Park intends to add a feedback mechanism telling DAMON about how effective it is being and adjusting parameters automatically. Someday, hopefully, it can be entirely self-tuning. 

There are a number of other features in the works as well. A mechanism to control reclaim via the virtio-balloon driver is planned, for example; that could provide better memory allocation on overcommitted hosts. 

Park would like to add an in-tree user-space tool for control of the system; this tool currently lived out-of-tree in a GitHub repository. Moving it in-tree would help users to understand how to manage the DAMON ABI, he said, and might also help developers avoid breaking that ABI. Dan Williams answered that moving a tool in-tree doesn't increase the flexibility of the ABI, and might well create more problems than it solves. It is easier to tie tooling to specific kernel versions when that tooling lives in the kernel tree — breaking out-of-tree users in the process, but that is exactly the kind of ABI break that must be avoided. Andrew Morton said that he would need to see a strong argument before he would accept an in-tree tool like that. 

Michal Hocko commented that he finds DAMON hard to understand; it is a complex memory-management system that lives outside of the kernel's memory-management subsystem. It would appreciate some good documentation that pulls it all together and shows how a use case might work. An article on "a certain Internet site" might also be beneficial, he said. 

Park had more future plans to discuss, but ran out of time in which to cover them. He has [posted his slides](<https://drive.google.com/file/d/10nsd4fq01On9p-oTmCS5KSW6qESmB6ER/view>) for those who are interested in looking further.
