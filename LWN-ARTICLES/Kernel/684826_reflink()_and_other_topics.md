---
title: reflink() and other topics
url: https://lwn.net/Articles/684826/
date: "April 26, 2016"
category: reflink
author: "By Jake Edge April 26, 2016 LSFMM 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 26, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

At the 2016 Linux Storage, Filesystem, and Memory-Management Summit, Darrick Wong led a session to discuss several features that he has been working on for XFS. While the session was slated as a plenary, the memory-management track was embroiled in another [discussion](<https://lwn.net/Articles/684300/>) so many of those developers were absent. Wong said that he had expected to stir up Dave Chinner with some of the topics, but Chinner ran into some [travel difficulties](<https://plus.google.com/108998708117877434524/posts/es7AU98mJGy>) and was unable to make it to the summit. 

Wong has implemented a reverse mapping of physical blocks to files and metadata for XFS. He had to change the btree that tracks that information in ways that broke many of the assumptions in the code. This is all in preparation for getting [`reflink()`](<https://lwn.net/Articles/333783/>) working for XFS. 

[ ![\[Darrick Wong\]](https://static.lwn.net/images/2016/lsf-wong-sm.jpg) ](<https://lwn.net/Articles/684885/>)

He has also been working on an [online scrubber for XFS](<https://www.spinics.net/lists/linux-ext4/msg51906.html>) that could be used by other filesystems. It would find and fix problems in the filesystem data structures. The idea is to make the scrubber "pluggable" so that it could sensibly deal with metadata and other differences between filesystems. The scrubber will walk the filesystem, locking directories and scrubbing their contents as it encounters them. He has run some "fairly trivial tests" of the scrubber on XFS and ext4. 

He has also been working on allowing two files to share pages in the page cache. Al Viro asked if that was for reading or writing; Wong said both would be supported. Chris Mason wondered why writing was in the mix. Wong said that the idea was to share pages as long as they aren't modified; a copy-on-write would be done if they are changed and both copies would be maintained at that point. 

Viro was concerned that the tracking of sections of files would need to be concerned with more operations than simply writes. For example, collapsing a range to shrink a file would need to be reflected in the page cache. Jan Kara said that the page cache entries for the file could just be invalidated when those operations are performed. It may be somewhat tricky to identify the pages of interest, but he thought it should be possible to make it all work. 

Wong said that he had come to the summit prepared to hear: "Gee, Darrick, you have a ten-year project on your hands". As it turned out, though, his impression is that sharing pages can probably be done, but there will be a lot of bookkeeping needed. 

Returning to `reflink()`, Wong said that he is trying to make the XFS interface as close as he can to what Btrfs does "to avoid splitting people's brains". He is also trying internally to get OCFS to use the same interface as well. Christoph Hellwig has been helping with testing and there are still some bugs in the btree code that need to be worked out. 

One question he had was how to handle quotas for reflinked files. An easy way to do it would be to charge the full size of the file at reflink time, but it might be better to wait until a copy-on-write happens. But Hellwig pointed out once a file is reflinked, the user loses control of when a copy-on-write operation might be done. Charging against the quota at that point could lead to situations where the quota was exceeded, so the full charge should be done at reflink time.
