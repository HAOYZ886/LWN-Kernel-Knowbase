---
title: "Predictive per-task write throttling"
url: https://lwn.net/Articles/152635/
date: "September 21, 2005"
category: "Memory management-Writeout throttling; Write throttling"
author: 
---

Memory-intensive tasks can be the bane of many a system administrator. One task which plows through vast numbers of pages can make the system thrash for everybody. The problem is especially acute when the memory hog is writing pages. Since each page dirtied by the process must be written to backing store before it can be reclaimed, a write-intensive task can quickly take a large portion of the system's memory out of commission. Often, a simple large file copy can noticeably impact a system's performance for some time after the copy apparently completes. 

The Linux VM subsystem attempts to address this problem with a simple form of write throttling. When the number of dirty pages gets too large, a process caught in the act of dirtying a page will be sent off to write out a few pages before being allowed to proceed. This technique slows the dirtying of pages while simultaneously helping to reclaim pages which have already been written to. This write throttling code makes no attempt to penalize any specific process, however; it will happily throttle any process which dirties a page at the wrong time. 

Andrea Arcangeli has decided to improve the situation with [a per-task predictive write throttling patch](<http://lwn.net/Articles/152277/>), currently found in the -mm tree. The patch is surprisingly simple - especially after noting that the bulk of it is involved with setting up the `/proc` and sysctl control interfaces. 

At its core, the patch adds a simple accumulator which keeps an approximate count of the number of pages dirtied by each process over the last five seconds. It then assumes that each process will continue to dirty pages at about the same rate into the future. The "are there too many dirty pages?" calculation is then changed to take this rate into account. The code, thus, is making a guess at what the dirty memory situation will be like in the future, based on what each process is doing. Any process which looks like it will cause too much memory to be dirtied gets to perform writeback for a while, while processes which are not writing to lots of pages are not given that particular chore. 

Andrea's preliminary results show that, with this patch in place, small, interactive tasks run in competition with a large copy task will run more quickly. Since the copy operation is being made to perform writeback (when it would have otherwise been dirtying more pages), more memory is available for the other tasks in the system. The interesting part of the result is that the copy task runs no slower with this patch in place. A process which is bound by the system's ability to write pages to disk will not benefit from being allowed to dirty the bulk of the system's memory, and it will not suffer by being throttled. So this little patch looks like it could be a winner for everybody involved.
