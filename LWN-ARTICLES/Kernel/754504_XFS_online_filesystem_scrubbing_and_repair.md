---
title: XFS online filesystem scrubbing and repair
url: https://lwn.net/Articles/754504/
date: "May 16, 2018"
category: "Filesystems-XFS"
author: "By Jake Edge May 16, 2018 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
May 16, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

In a filesystem track session at the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Darrick Wong talked about the online scrubbing and repair features he has been working on. His target has mostly been XFS, but he has concurrently been working on scrubbing for ext4. Part of what he wanted to discuss was the possibility of standardizing some of these interfaces across different filesystem types. 

Filesystem scrubbing is typically an ongoing activity to try to find corrupted data by periodically reading the data on the disk. Online repair attempts to fix the problems found by using redundant information (or metadata that can be calculated from other information) stored elsewhere in the filesystem. As described in [Wong's patch series](<https://lwn.net/Articles/752182/>), both scrubbing and repair are largely concerned with filesystem metadata, though scrubbing data extents (and repairing them if possible) is also supported. Wong said that XFS now has online scrubbing support, but does not quite have the online repair piece yet. 

[ ![\[Darrick Wong\]](https://static.lwn.net/images/2018/lsf-wong-sm.jpg) ](<https://lwn.net/Articles/754532/>)

Btrfs has support for online scrubbing and ext4 will eventually as well. Wong wondered if there was an opportunity to create a common wrapper for user space. Ted Ts'o said that it would help if there was some clarity about the goals and requirements of a scrubber tool. He asked, is it a cron job that scrubs all the filesystems or might there be individual crontab entries for ext4 and XFS? Clearly the goal should be to make the system administrator's life better. 

Chris Mason brought up the CRC checks that the filesystems currently do. When those CRC checks fail, each filesystem logs its own message to dmesg. There is no consistency between the filesystems for that message. Wong recommended that Btrfs return a "filesystem corrupt" error status to user space as ext4 and XFS do, but Mason pointed out that CRC errors are not only found during a filesystem scrubbing. 

Kent Overstreet said that he had a framework that could be used for long-running jobs in the kernel. It returns a file descriptor that can be used to monitor the job. Wong said that the XFS scrubbing consists of many `ioctl()` commands that are called from user space. Overstreet said that sounded harder to deal with. Josef Bacik said that Btrfs is similar to XFS, but that having a single file descriptor might be better. 

Dave Chinner wondered if there was a way to have a single scrubbing command that handled any kind of filesystem, so that users do not have to remember how to do it for each type. No one seemed opposed to the idea but getting there may take some time. 

When data errors are found, some users may not really want to have the filesystem try to repair things, Ric Wheeler said. Instead they will just want the name of the file containing the error so that they can simply get a copy from another server. That requires mapping the blocks back to a path. He also said that a recent paper showed that, while SSDs will last a lot longer than rotating storage, they will generate many more errors (on the order of 10-15 times more) than rotating storage over that time. So these kinds of problems will become more prevalent. 

Another thing that needs to be standardized is the I/O priority that these scanners will run with, Mason said. 

Wong suggested starting with a simple common scrubbing wrapper that would do the right thing for each filesystem type. It would just report whether the metadata had errors and whether the data had errors. From that, administrators could then decide how to fix the errors. Chinner said that there needs to be some standard on what errors get returned, but Wong suggested starting with something simple: 0 for OK, 1 to indicate a problem and that the administrator should check the logs for more information. It was generally agreed that would be a reasonable place to start, though Ts'o cautioned there would be a need to eventually standardize more pieces at multiple levels.
