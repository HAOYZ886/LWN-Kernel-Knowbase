---
title: "Filesystems: chunkfs and reiser4"
url: https://lwn.net/Articles/231585/
date: "April 24, 2007"
category: "Filesystems-Chunkfs; Filesystems-Reiser4"
author: 
---

One of the fundamental problems facing filesystem developers is that, while disks are getting both larger and faster, the rate at which they are growing exceeds the rate at which they are speeding up. As a result, the time required to read an entire disk is growing. There is little joy in waiting for a filesystem checker to do its thing during a system reboot, so the prospect of ever-longer fsck delays is understandably lacking in appeal. Unfortunately, that is the direction in which things are going. Journaling filesystems can help avoid fsck, but only in situations where the filesystem has not suffered any sort of corruption. 

Given that filesystem checks are something we have to deal with, it's worth thinking about how we might make them faster in the era of terabyte disks. One longstanding idea for improving the situation was recently posted in the form of [chunkfs](<http://lwn.net/Articles/231482/>), "fs fission for faster fsck." The core idea is to take a filesystem and split it into several independent filesystems, each of which maintains its own clean/dirty state. Should things go wrong, only those sub-filesystems which were active at the time of failure need to be checked. 

Like many experimental filesystem developments, chunkfs is built upon ext2. Internally, it is a series of separate ext2 filesystems which look like a single system to the higher layers of the filesystem. Each chunk can be maintained independently by the filesystem code, but the individual chunks are not visible outside of the filesystem. The idea is relatively simple, though, as always, there are a few pesky details to work out. 

One is that inode numbers in the larger chunkfs filesystem must be unique. Each chunk, however, maintains its own list of inodes starting with number one, so inode numbers will be reused from one chunk to the next. Chunkfs makes these numbers unique by putting the chunk number in the upper eight bits of every inode number. As a result, there is a maximum of 256 chunks in any chunkfs filesystem. 

A trickier problem comes about when a file grows. The filesystem will try to allocate additional file blocks in the chunk where the file was originally created. Should that chunk fill up, however, something else needs to happen; it would not be good for the filesystem to return "no space" errors when free space does exist in other chunks. The answer here is the creation of a "continuation inode." These inodes track the allocation of blocks in a different chunk; they look much like files in their own right, but they are part of a larger array of block allocations. The "real" inode for a given file can have pointers to up to four continuation inodes in different chunks; if more are needed, each continuation inode can, itself, point to another four continuations. Thus, continuation inodes can be chained to create files of arbitrary length. 

This code is in a relatively early state; the text with the patch notes that ""this is a preliminary implementation and lots of changes are expected before this code is even sanely usable."" There is a set of tools which can be used by people who would like to test out chunkfs filesystems with well backed-up data. With some care and some testing, chunkfs may grow to the point that it's stable and shortening fsck times worldwide. 

Meanwhile, one of the longest stories in Linux filesystem development has to be the reiser4 filesystem. By the time Hans Reiser [first asked for the merging of reiser4](<http://lwn.net/Articles/41653/>) in July, 2003, the filesystem had been under development for some years. Almost four years have passed since then, and reiser4 remains outside of the mainline kernel. Hans Reiser is now out of the picture, his company (Namesys) is in trouble, and, to a casual observer, reiser4 appears not to be going anywhere. 

There has been a recent increase in interest in this filesystem, though. It [turns out](<https://lwn.net/Articles/231702/>) that two Namesys employees are still working on the filesystem "mostly on enthusiasm." They have been feeding patches through to the -mm tree, and they are getting toward the end of their list of things to fix. So we might see a new push for inclusion of reiser4, perhaps as soon as 2.6.23. But, [says Andrew Morton](<https://lwn.net/Articles/231703/>), some things would have to happen; in particular, there needs to be a new review of the reiser4 code. 

To get it unstuck we'd need a general push, get people looking at and testing the code, get the vendors to have a serious think about it, etc. We could do that - it'd require that the namesys people (and I) start making threatening noises about merging it, I guess. 

Or we could move all the reiser4 code into kernel/sched.c - that seems to get people fired up. 

Your editor will go out on a limb and suggest that a mass move of the reiser4 code is unlikely. But a new round of talk on actually merging this filesystem is starting to look reasonably likely. There's enough work - and enough interesting ideas - in this code that people are unwilling to let it just fade away. Perhaps, soon, it will be heading for its long-sought spot in the mainline.
