---
title: "Some buffer-heads cleanup work"
url: https://lwn.net/Articles/1077767/
date: "June 17, 2026"
category: "Block layer-Buffer heads"
author: "By Jake Edge June 17, 2026 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 17, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Jan Kara has been [working on cleaning up](<https://lwn.net/ml/all/20260326082428.31660-1-jack@suse.cz/>) how [buffer heads](<https://www.kernel.org/doc/html/v7.1-rc7/filesystems/buffer.html>) are used by some kernel filesystems. In a short filesystem-track session at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), he gave an update on that work and where it is headed. Topics included generic infrastructure to track buffer heads for metadata, a buffer-head cleanup for the Amiga filesystem, and some planned locking fixes. 

Buffer heads are ""ancient stuff"", he began, having been part of the kernel ""basically since day zero of Linux"". They are used to track filesystem information at the granularity of blocks, rather than folios. Kernel filesystem developers are trying to remove buffer heads from the data path in filesystems, but they are still used in the metadata path for many filesystems. Overall, buffer heads are not going away anytime soon because those filesystems need fine-grained tracking for the state of individual blocks. 

[ ![\[Jan Kara\]](https://static.lwn.net/images/2026/lsfmb-kara-sm.png) ](<https://lwn.net/Articles/1078069/>)

One of the things he has been working on is generic infrastructure for tracking all of the metadata blocks that belong to a given inode so that they can be flushed on an [`fsync()`](<https://man7.org/linux/man-pages/man2/fsync.2.html>) call. That infrastructure is used by ext4, ext2, [UDF](<https://docs.kernel.org/filesystems/udf.html>), [VFAT](<https://docs.kernel.org/filesystems/vfat.html>), and a few others, he said. He factored the metadata-buffer-head tracking out of the generic inode structure and into the filesystem-private part of the inode that can be used by filesystems that care. Filesystems that do not need that tracking can have an inode that is 40 bytes smaller, he said. That work has been merged by Christian Brauner for the 7.1 kernel. 

Kara also made a [small cleanup](<https://lwn.net/ml/all/20260525085821.769119-11-jack@suse.cz/>) for the [Amiga Fast File System](<https://en.wikipedia.org/wiki/Amiga_Fast_File_System>) (AFFS) [Linux implementation](<https://docs.kernel.org/filesystems/affs.html>). The filesystem took the trouble to track the metadata buffer heads, but never used that information at `fsync()` time. Since the maintainer told him that AFFS performance is not really a concern, he removed the tracking instead of switching AFFS to use the new infrastructure. 

There is a race in the tracking of the metadata buffer heads that can result in the inode and all of the metadata not being written to the backing store. So if an `fsync()` is followed by a crash, the metadata that should have been flushed to disk may be missing. That has been worked around for ext4, but all of the other filesystems using the new infrastructure are vulnerable to it. He is working on a generic fix, which will require expanding the structure used to track the metadata buffer heads. 

He is also planning to rework the locking for buffer heads. There are two locks that protect buffer heads when they are attached to folios, he said; one is the folio lock ([`folio_lock()`](<https://elixir.bootlin.com/linux/v7.1/source/include/linux/pagemap.h#L1133>)) for the folio it is attached to and the other is the private lock for the mapping (`i_private_lock` in [`struct address_space`](<https://elixir.bootlin.com/linux/v7.1/source/include/linux/fs.h#L453>)). The latter is used in places where the folio lock, which can sleep, cannot be taken, but he would like to stop using the private lock because it substantially complicates the locking; he would like to use read-copy-update (RCU) instead. He hopes to get that work done over the next year or less. 

Christoph Hellwig asked about an ""only vaguely related"" problem where ext4 in [`data=journal` mode](<https://www.kernel.org/doc/html/latest/filesystems/ext4/journal.html>) can create dirty buffer heads that are detached from the mapping and ""need magic handling"". He wondered if Kara had any ideas on how to untangle that. Kara said that the problem can occur in other modes, but is more common with `data=journal`. It happens when the VFS would like to reclaim a block or folio, but the filesystem will not allow that to be done because it is still journaling the data, which puts the data into ""a strange limbo state"". 

He has some ideas on what needs to be done. There are two paths where journaled buffer heads undergo writeback; one is the standard path that ends up at the block layer and works fine, but the other is in the journal path that simply writes the blocks tracked by the buffer heads without changing the state of the folios that contain them, which creates the problem. The fix is for the journal path to use the standard writeback machinery to write folios ""instead of stealing the buffer heads from underneath"", which ""is a bit non-trivial of a rewrite of the journaling machinery"", he said to some knowing laughter. He can provide pointers and some code to anyone who wants to tackle the problem. 

Hellwig said that he has been working with Namjae Jeon on [converting the exfat filesystem to use iomap](<https://lwn.net/ml/all/20260518114705.9601-1-linkinjeon@kernel.org/>) for its data path. Since exfat is a ""typical simple filesystem"", that work could provide a good template to convert other filesystems to use [iomap](<https://docs.kernel.org/filesystems/iomap>) in a similar way, ""because it's a recent conversion of a generic doesn't-do-anything-crazy filesystem"". That work originally targeted the 7.1 merge window but ran into some problems; it should appear in 7.2. With no more topics to discuss, the session concluded.
