---
title: "Controlling block-I/O latency"
url: https://lwn.net/Articles/753277/
date: "May 3, 2018"
category: "Control groups-IO bandwidth controllers"
author: "By Jake Edge May 3, 2018 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 3, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Chris Mason and Josef Bacik led a brief discussion on the block-I/O controller for control groups (cgroups) in the filesystem track at the 2018 Linux Storage, Filesystem, and Memory-Management Summit. Mostly they were just aiming to get feedback on the approach they have taken. They are trying to address the needs of their employer, Facebook, with regard to the latency of I/O operations. 

Mason said that the goal is to strictly control the latency of block I/O operations, but that the filesystems themselves have priority inversions that make that difficult. For Btrfs and XFS, they have patches to tag the I/O requests, which mostly deals with the problem. They have changes for ext4 as well, but those are not quite working yet. 

[ ![\[Josef Bacik\]](https://static.lwn.net/images/2018/lsf-bacik-sm.jpg) ](<https://lwn.net/Articles/753392/>)

Bacik said the current block-I/O controller does not work for the company's use case. Facebook wants to be able to specify a latency target for a cgroup; if at any point that target is being exceeded, other cgroups should have their I/O throttled. The throttling is done by reducing the amount of I/O that is allowed to be in-flight for the other groups. 

Kent Overstreet asked why this isn't done in an I/O scheduler. Bacik said Facebook wants to protect a certain workload, at the expense of any others. Mason noted that the workloads are already put together using cgroups, so there is no reason to create an I/O scheduler. Dave Chinner said that the use case is only concerned with throttling, not scheduling. 

There is an issue of throttling filesystem-initiated I/Os for metadata and swap, Bacik said. The code is inserting delays into those in order to throttle that I/O when needed. That code is not yet present in XFS, Mason said; it was simply poked into Btrfs for testing. 

Jan Kara said that this code should probably only be used with the no-op scheduler or another simple I/O scheduler. That is what is recommended for XFS anyway, Chinner said. Bacik said that CFQ is not used at Facebook, even on spinning disks, as it will cause latency spikes for no apparent reason. He said that Facebook wants to be able to use writeback throttling together with latency throttling; it is not working correctly at the moment, but was earlier, so he will figure it out and fix it. 

There are actually two separate use cases; one is the protected workload, but the other is for shared workloads. In the latter case, both the maximum latency and maximum I/O rate settings will be used. The latter will be for setting expectations, Mason said. If you end up giving 100MB per second most of the time, people will come to expect that rate and applications will fail when it occasionally drops from there. But if you always give 20MB per second, the applications will scale their I/O to accommodate that. 

Hearing no major objections to the idea, Bacik said he would post patches in a week or two after the summit. Chinner asked his usual question about tests; Bacik said that he had some scripts that he wrote in xfstests style. He will add those tests to the patch set.
