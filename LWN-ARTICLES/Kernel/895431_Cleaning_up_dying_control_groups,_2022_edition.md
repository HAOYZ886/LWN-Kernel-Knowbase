---
title: "Cleaning up dying control groups, 2022 edition"
url: https://lwn.net/Articles/895431/
date: "May 19, 2022"
category: "Control groups; Memory management-Control groups"
author: "By Jonathan Corbet May 19, 2022 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 19, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Control groups are a useful system-management feature, but they can also consume a lot of resources, especially if they hang around on the system after they have been deleted. Roman Gushchin [described the problems that can result](<https://lwn.net/Articles/787614/>) at the 2019 Linux Storage, Filesystem, Memory-management and BPF Summit (LSFMM); he returned during the [2022 LSFMM](<https://events.linuxfoundation.org/lsfmm/>) to revisit the issue, especially as it relates to the memory controller. Progress has been made, but the problem is not yet solved. 

Modern systems, he began, can create and destroy vast numbers of control groups, especially if they are running systemd. The cost of creating a control group is low, but the destruction costs can be "brutal". Sometimes, the task of getting rid of an old control group never completes, leaving the system paying the cost of having a large number of dying control groups sitting around. [![\[Roman Gushchin\]](https://static.lwn.net/images/conf/2022/lsfmm/RomanGushchin-sm.png)](<https://lwn.net/Articles/895435/>)

There are a number of difficulties involved in cleaning up a control group. If the memory controller is in use, the group cannot be deleted until the pages charged to it are reclaimed, and that is a costly process. The [`mem_cgroup` structure](<https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/memcontrol.h#L227>) used to represent a memory control group is large; it can occupy hundreds of kilobytes of space. On a large system, the amount of memory consumed by these structures can reach into the gigabyte range. These are old problems, he said, but they are still with us. 

The problem is exacerbated by the inability to quickly find the memory that is charged to any given control group; there are statistics but otherwise the kernel has little visibility in this area, Gushchin said. Even worse, though, is when memory is shared between control groups. Then the system probably has living groups using resources that were created by (and are charged to) dying groups; the accounting will not be correct in this case. In general, the kernel has never handled memory shared between groups well; the first group to create any given page is charged for it. In a typical system, much of the working set will "belong" to older control groups; that messes up the statistics and prevents usage limits from working properly. 

Some work has been done, he said, including a lot of plain fixes and optimizations. Slab reparenting, which he had described in 2019, has helped a lot by eliminating the problem of old groups being pinned by remaining slab-allocated objects. Slab accounting has been reworked in general, providing byte-resolution charging and reparenting; this work is being extended beyond the slab layer. Writeback of memory belonging to control groups has been cleaned up; it had been holding references that could keep an old group around. Statistics from the memory controller have been improved in general. 

The biggest remaining question, he said, is what to do with the page cache. Memory in the page cache gets left behind when a control group exits. There is [a reparenting patch set](<https://lwn.net/ml/linux-mm/20220216115132.52602-1-songmuchun@bytedance.com/>) from Muchun Song in circulation, but Gushchin is not sure that the approach is correct. He wondered if reparenting page-cache pages makes sense, or whether page-cache pages need to hold a reference to the control group to which they are charged at all. There is also a patch from Waiman Long to force the early release of per-CPU memory, but Gushchin described it as a "band-aid" that adds more complexity. He mentioned, instead, the possibility of marking leftover page-cache pages with a special flag that would cause them to be charged to the next user that came along. 

At another level, there is work being done in systemd to end the practice of creating and deleting control groups; that work may land soon, Gushchin said. Relying on that change is questionable, though, since it's delegating the problem to user space. 

The session wound down without a lot of discussion. Johannes Weiner did remark, though, that the problem needs to be solved even if systemd changes to avoid triggering it. The problem will continue to pop up until it is fixed in the right place.
