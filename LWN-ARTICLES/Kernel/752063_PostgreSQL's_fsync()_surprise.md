---
title: "PostgreSQL's fsync() surprise"
url: https://lwn.net/Articles/752063/
date: "April 18, 2018"
category: "Block layer-Error handling"
author: "By Jonathan Corbet April 18, 2018"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 18, 2018

Developers of database management systems are, by necessity, concerned about getting data safely to persistent storage. So when the PostgreSQL community found out that the way the kernel handles I/O errors could result in data being lost without any errors being reported to user space, a fair amount of unhappiness resulted. The problem, which is exacerbated by the way PostgreSQL performs buffered I/O, turns out not to be unique to Linux, and will not be easy to solve even there. 

Craig Ringer first [reported the problem](<https://lwn.net/Articles/752093/>) to the pgsql-hackers mailing list at the end of March. In short, PostgreSQL assumes that a successful call to `fsync()` indicates that all data written since the last successful call made it safely to persistent storage. But that is not what the kernel actually does. When a buffered I/O write fails due to a hardware-level error, filesystems will respond differently, but that behavior usually includes discarding the data in the affected pages and marking them as being clean. So a read of the blocks that were just written will likely return something other than the data that was written. 

What about error status reporting? One year ago, the Linux Filesystem, Storage, and Memory-Management Summit (LSFMM) included [a session](<https://lwn.net/Articles/718734/>) on error reporting, wherein it was described as "a mess"; errors could easily be lost so that no application would ever see them. [Some patches](<https://lwn.net/Articles/724307/>) merged during the 4.13 development cycle improved the situation somewhat (and 4.16 had some changes to improve it further), but there are still ways for error notifications to be lost, as will be described below. If that happens to a PostgreSQL server, the result can be silent corruption of the database. 

PostgreSQL developers were not pleased. Tom Lane [described](<https://lwn.net/Articles/752096/>) it as ""kernel brain damage"", while Robert Haas [called it](<https://lwn.net/Articles/752097/>) ""100% unreasonable"". In the early part of the discussion, the PostgreSQL developers were clear enough on what they thought the kernel's behavior should be: pages that fail to be written out should be kept in memory in the "dirty" state (for later retries), and the relevant file descriptor should be put into a permanent error state so that the PostgreSQL server cannot miss the existence of a problem. 

#### Where things go wrong

Even before the kernel community came into the discussion, though, it started to become clear that the situation was not quite as simple as it might seem. Thomas Munro [reported](<https://lwn.net/Articles/752098/>) that Linux is not unique in behaving this way; OpenBSD and NetBSD can also fail to report write errors to user space. And, as it turns out, the way that PostgreSQL handles buffered I/O complicates the picture considerably. 

That mechanism was [described in detail](<https://lwn.net/Articles/752101/>) by Haas. The PostgreSQL server runs as a collection of processes, many of which can perform I/O to the database files. The job of calling `fsync()`, however, is handled in a single "checkpointer" process, which is concerned with keeping on-disk storage in a consistent state that can recover from failures. The checkpointer doesn't normally keep all of the relevant files open, so it often has to open a file before calling `fsync()` on it. That is where the problem comes in: even in 4.13 and later kernels, the checkpointer will not see any errors that happened before it opened the file. If something bad happens before the checkpointer's `open()` call, the subsequent `fsync()` call will return successfully. There are a number of ways in which an I/O error can happen outside of an `fsync()` call; the kernel could encounter one while performing background writeback, for example. Somebody calling `sync()` could also encounter an I/O error — and consume the resulting error status. 

Haas described this behavior as failing to live up to what PostgreSQL expects: 

What you have (or someone has) basically done here is made an undocumented assumption about which file descriptors might care about a particular error, but it just so happens that PostgreSQL has never conformed to that assumption. You can keep on saying the problem is with our assumptions, but it doesn't seem like a very good guess to me to suppose that we're the only program that has ever made them. 

Joshua Drake eventually [moved the conversation](<https://lwn.net/Articles/752103/>) over to the ext4 development list, bringing in part of the kernel development community. Dave Chinner quickly [described](<https://lwn.net/Articles/752104/>) this behavior as ""a recipe for disaster, especially on cross-platform code where every OS platform behaves differently and almost never to expectation"". Ted Ts'o, instead, [explained](<https://lwn.net/Articles/752105/>) why the affected pages are marked clean after an I/O error occurs; in short, the most common cause of I/O errors, by far, is a user pulling out a USB drive at the wrong time. If some process was copying a lot of data to that drive, the result will be an accumulation of dirty pages in memory, perhaps to the point that the system as a whole runs out of memory for anything else. So those pages cannot be kept if the user wants the system to remain usable after such an event. 

Both Chinner and Ts'o, along with others, said that the proper solution is for PostgreSQL to move to direct I/O (DIO) instead. Using DIO gives a greater level of control over writeback and I/O in general; that includes access to information on exactly which I/O operations might have failed. Andres Freund, like a number of other PostgreSQL developers, has [acknowledged](<https://lwn.net/Articles/752107/>) that DIO is the best long-term solution. But he also noted that getting there is ""a metric ton of work"" that isn't going to happen anytime soon. Meanwhile, he [said](<https://lwn.net/Articles/752108/>), there are other programs (he mentioned `dpkg`) that are also affected by this behavior. 

#### Toward a short-term solution

As the discussion went on, a fair amount of [attention](<https://lwn.net/Articles/752110/>) was paid to the idea that write failures should result in the affected pages being kept in memory, in their dirty state. But the PostgreSQL developers had quickly moved on from that idea and were not asking for it. What they really need, in the end, is a reliable way to know that something has gone wrong. Given that, the normal PostgreSQL mechanisms for dealing with errors can take over; in its absence, though, there is little that can be done. 

One idea that came up a few times was to respond to an I/O error by marking the file itself (in the inode) as being in a persistent error state. Such a change, though, would take Linux behavior further away from what POSIX mandates and would raise some other questions, including: when and how would that flag ever be cleared? So this change seems unlikely to happen. 

At one point in the discussion, Ts'o [mentioned](<https://lwn.net/Articles/752112/>) that Google has its own mechanism for handling I/O errors. The kernel has been instrumented to report I/O errors via a netlink socket; a dedicated process gets those notifications and responds accordingly. This mechanism has never made it upstream, though. Freund [indicated](<https://lwn.net/Articles/752113/>) that this kind of mechanism would be ""perfect"" for PostgreSQL, so it may make a public appearance in the near future. 

Meanwhile, Jeff Layton [pondered](<https://lwn.net/Articles/752114/>) another idea: setting a flag in the filesystem superblock when an I/O error occurs. A call to `syncfs()` would then clear that flag and return an error if it had been set. The PostgreSQL checkpointer could make an occasional `syncfs()` call as a way of polling for errors on the filesystem holding the database. Freund [agreed](<https://lwn.net/Articles/752115/>) that this might be a viable solution to the problem. 

Any such mechanism will only appear in new kernels, of course; meanwhile, PostgreSQL installations tend to run on old kernels maintained by enterprise distributions. Those kernels are likely to lack even the improvements merged in 4.13. For such systems, there is little that can be done to help PostgreSQL detect I/O errors. It may come down to running a daemon that scans the system log, looking for reports of I/O errors there. Not the most elegant solution, and one that is complicated by the fact that different block drivers and filesystems tend to report errors differently, but it may be the best option available. 

The next step is likely to be a discussion at the 2018 LSFMM event, which happens to start on April 23\. With luck, some sort of solution will emerge that will work for the parties involved. One thing that will not change, though, is the simple fact that error handling is hard to get right.
