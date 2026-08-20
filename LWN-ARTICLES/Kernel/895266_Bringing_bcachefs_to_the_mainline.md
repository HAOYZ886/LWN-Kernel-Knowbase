---
title: Bringing bcachefs to the mainline
url: https://lwn.net/Articles/895266/
date: "May 17, 2022"
category: "Filesystems-bcachefs"
author: "By Jake Edge May 17, 2022 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 17, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

[Bcachefs](<https://bcachefs.org/>) is a longstanding out-of-tree filesystem that grew out of the [bcache caching layer](<https://lwn.net/Articles/394672/>) that has been in the kernel for nearly ten years. Based on a session led by Kent Overstreet at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), though, it would seem that bcachefs is likely to be heading upstream soon. He intends to start the process toward mainline inclusion over the next six months or so. 

Overstreet is often asked what the target use cases for bcachefs are; "the answer is everything". His longstanding goal is to be "reliable and robust enough to be the XFS replacement". It has been a few years since he last [gave an update](<https://lwn.net/Articles/755276/>) at LSFMM, so he began by listing the features and changes that have been added. 

[ ![\[Kent Overstreet\]](https://static.lwn.net/images/2022/lsfmm-overstreet-sm.png) ](<https://lwn.net/Articles/895446/>)

Support for [reflinks](<https://lwn.net/Articles/333783/>), effectively copy-on-write (COW) links for files, has been added to bcachefs. After that support was added, Dave Chinner asked him about snapshots; he had been avoiding implementing snapshots but some reworking that he did on how bcachefs handles extents made it easier to do so. He added snapshot support and there are no scalability issues; he has done up to a million snapshots on test virtual machines without any problems. Snapshots in bcachefs have the same external interface as Btrfs (i.e. subvolumes), though the internal implementation is different. 

More recently, the bcachefs allocator has been rewritten. Bcache, which is the ancestor of bcachefs, had some "algorithmic scalability issues" because it was created in the days where SSDs were around 100GB in size. But he has bcachefs users on 50TB arrays; things that work fine for the smaller sizes do not scale well, he said. So he has been reworking various pieces of bcachefs to address those problems. 

There are now persistent data structures for holding data that used to require the filesystem to periodically "walk the world" by scanning the filesystem structure. Backpointers have been added so that data blocks point to the file that contains them, which is important to accelerate the "copygc" operation. That operation does a form of garbage collection, but it (formerly) required scanning through the filesystem structure. He said that it is also important for [supporting zoned storage devices](<https://lwn.net/Articles/788851/>), which is still a little ways off but is coming. 

#### Merging

Overstreet wants to be able to propose bcachefs for upstream inclusion "but not go insane and still be able to write code when that happens". The [to-do list](<https://www.evilpiepirate.org/~kent/.plan.txt>) is always expanding, but the "really big pain points" have mostly been dealt with at this point. There is good reason to believe that upstreaming is close, he said. 

Amir Goldstein asked about where and how bcachefs is being used in production now. Overstreet said that he knows it is being used, but he does not know how many sites are using it. He generally finds out when someone asks him to look at a problem. Bcachefs is mostly used by video production companies that need to deal with multiple 4K streams for editing multi-camera setups, he said; they have been using it for several years now. Bcachefs was chosen because it had better performance than Btrfs for those workloads and, at the time, was the only filesystem with certain features that were needed. 

Josef Bacik said that he looked at the to-do list and noted that it was mostly bcachefs-internal items. He said that the goal when bcachefs was discussed at LSFMM in 2018 was to get the interfaces to the rest of Linux into good shape, since that would be the focus of any mailing-list review. None of the other filesystem developers know much about the internals of bcachefs, so they would not be able to review that code directly. He wondered what was left to do before the upstream process could begin. 

Overstreet said that the `ioctl()` interface was one of the things discussed, but it has not changed in a while. He is more concerned about ensuring that the on-disk format changes are settling down. He had been pushing out those kinds of changes fairly frequently, and the backpointer support requires another, but after that, he does not see any other changes of that sort on the horizon. 

Bacik asked how much more work Overstreet wanted to do internally before he would be ready to start talking about merging bcachefs and what was holding it back. Bacik also wanted to know what Overstreet needed from other filesystem developers as part of that process. The biggest thing holding him back, Overstreet said, is that he wants to be able to respond to all of the bug reports that will arise when there are lots more users of bcachefs. So he wants to make sure that the bigger development projects get taken care of before he gets to that point. 

He said that it is far faster for him to fix a bug when he finds it himself, rather than having to figure out a way to reproduce a problem that someone else has found. So he is hoping to get rid of as many bugs as he can before merging. That process has been improved greatly by the debugging support he added to bcachefs over the last few years; over the last six months, he said, that effort "has been paying off in a big way". For example, the allocator rewrite went smoothly because of those tools. 

Much of that revolves around the [printbuf mechanism](<https://lwn.net/Articles/892611/>) that he recently proposed for the kernel. That work came out of his interest in getting better logging information for bcachefs. There are "pretty printers" for various bcachefs data structures and their output can be logged. He is now able to debug using `grep`, rather than a debugger for many of the kinds of problems he encounters. He said that he would be talking more about that infrastructure in a [memory-management session](<https://lwn.net/Articles/894546/>) the next day. 

#### Wart sharing

Chris Mason said that he had a question along the lines of those from Bacik, but "a lot more selfish". Btrfs has a lot of warts in how it interfaces with the virtual filesystem (VFS) layer, in part because its inode numbers are effectively huge, but also due to various `ioctl()` commands for features like reflink. He is looking forward to some other filesystem coming into Linux that is "sharing my warts"; that may lead to finding better ways to solve some of those problems, he said. 

Overstreet said that bcachefs has the same basic problem that Btrfs does with regard to [inode numbers, subvolumes, and NFS](<https://lwn.net/Articles/866582/>); he has not spent a lot of time thinking about it but would like to use the Btrfs solution once one is found. Mason said that every eight months or so, someone comes along to say that the problem is stupid and easy to fix, then the Btrfs developers have to show once again that the problem is stupid, but hard to fix. Bacik agreed that a second filesystem with some of the same kinds of problems will help; it is difficult to make certain kinds of changes because there "seems to be an allergic reaction" to interface changes that are only aimed at Btrfs problems. 

Ted Ts'o had two suggestions for Overstreet; first, before adding a whole lot of new users, some kind of bcachefs repair facility is probably necessary. Overstreet said that part was all taken care of. Ts'o also said that having an automated test runner that exercised various different bcachefs configuration options would be useful. He has a test harness, and Luis Chamberlain has a different one, either of which would probably serve the needs of bcachefs. Bacik noted that there is a slot later in LSFMM to discuss some of that. 

Overstreet returned to the subject of debugging tools, as it is "the thing that excites me the most". The pretty-printer code is shared by both kernel and user space, which makes it easier to find problems, he said. `grep` is his tool of choice for finding problems, even for difficult things like deadlocks. He demonstrated some of the kinds of information he could extract using those facilities. 

Mason suggested looking into integrating this work into the [drgn kernel debugger](<https://github.com/osandov/drgn>), which was the subject of a [session at LSFMM 2019](<https://lwn.net/Articles/789641/>). It is a Python-based, live and post-crash kernel debugger that is used extensively at Facebook; every investigation of a problem in production starts by poking around using the tool. Bacik agreed, noting that drgn allows writing programs that can step through data structures in running systems to track down a wide variety of filesystem (and other) problems. Overstreet said that he would be looking into it. 

Overstreet pointed to the [bcachefs: Principles of Operation](<https://bcachefs.org/bcachefs-principles-of-operation.pdf>) document as a starting point for user documentation. It is up to 25 pages at this point, organized by feature, and will be getting fleshed out further soon. 

While Overstreet's hesitance to push for merging bcachefs is understandable, Bacik said, he and others have some selfish reasons for wanting to see that happen. He said he did not want to rush things, but did Overstreet have a timeline? Overstreet said that he would like to see it happen within the next six months. Based on the recent bug reports, he thinks that is a realistic goal. 

Goldstein wondered when the Rust rewrite would be coming. Overstreet said that there is already some user-space Rust code in the repository; as soon as Rust support lands in the kernel, he would like to make use of it. There are "so many little quality-of-life improvements in Rust", such as proper iterators rather than "crazy for-loop macros". Bacik said that many were waiting for that support in the kernel; Overstreet suggested that those who are waiting be a bit noisier to make it clear that there is demand for it. With that, time expired on the session, but it seems we may see bcachefs and Rust racing to see which can land in the kernel first.
