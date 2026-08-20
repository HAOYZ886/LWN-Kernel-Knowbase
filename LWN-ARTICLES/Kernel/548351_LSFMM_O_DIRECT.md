---
title: "LSFMM: O_DIRECT"
url: https://lwn.net/Articles/548351/
date: "May 1, 2013"
category: "Block layer-Direct IO"
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Dave Chinner and Josef Bacik led a short discussion at the 2013 LSFMM Summit about the future of direct I/O (i.e. `O_DIRECT`) for "advanced filesystems". Bacik noted that he had to add "hacks on hacks" to make direct I/O work and "not suck performance-wise" for Btrfs. While he was doing that work, he wondered if it made sense to replace the existing generic direct I/O code with something new that would work for Btrfs and others. 

Ted Ts'o noted that there are only half-a-dozen filesystems that support direct I/O, so a "v2" of the generic direct I/O code "would be great". It should do what is necessary for XFS and Btrfs, he said, and ext4 could eventually migrate to it. No one would want to retrofit ext3, though, so the existing direct I/O code would likely need to continue to exist, he said. 

For XFS, doing a `writev()` with 256 4K chunks of data via the generic code causes 256 separate calls to `get_block()` (which is used by direct I/O to map blocks), Chinner said. When he changed XFS to do it "all in one go", it caused crashes in xfstests. Basically, direct I/O has been optimized such that it can't be changed any more, he said. 

Bacik agreed, saying that he had seen similar problems with Btrfs. It was agreed that the existing code is also quite complex. It "works, but I can't debug it", Chinner said. Joel Becker, Zach Brown, and others agreed. 

Becker noted that the consensus seems to be that coming up with something new is the right approach. James Bottomley asked who would be doing the work, and Becker said that he would. There was some discussion of "wish lists" of features. It would be nice to get rid of the existing code, perhaps by rewriting it in terms of the new generic direct I/O code, but no one is interested in changing ext3 and JFS to directly use a new interface. Becker said that he had a 500-600 line patch to "deal with generic direct I/O"; he will send those patches soon, he said, to a round of applause.
