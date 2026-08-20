---
title: Inserting a hole into a file
url: https://lwn.net/Articles/629965/
date: "January 21, 2015"
category: fallocate
author: "By Jake Edge January 21, 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
January 21, 2015

Last March, we [looked](<https://lwn.net/Articles/589260/>) at a proposal for a new `fallocate()` option to collapse a range of blocks within a file. The `FALLOC_FL_COLLAPSE_RANGE` flag was added to the 3.15 kernel; its counterpart, `FALLOC_FL_INSERT_RANGE`, has been [proposed](<https://lwn.net/Articles/629365/>) by the same developer: Namjae Jeon. It would provide a way to open up a range of blocks within a file, without requiring an expensive data copy. 

The example use case that Jeon has used for both new flags is the removal (using `FALLOC_FL_COLLAPSE_RANGE`) or insertion (using `FALLOC_FL_INSERT_RANGE`) of advertisements into large video files. While that particular example may not resonate with everyone, there are other uses for quickly removing and inserting chunks of data in the middle of large files. For example, doing non-linear editing on various types of media (video, in particular) may benefit from reducing the amount of data copying needed. The requirement that the ranges be block-aligned, though, could limit the overall usefulness of both flags. 

The [`fallocate()`](<http://man7.org/linux/man-pages/man2/fallocate.2.html>) system call provides a means for programmers to alter the allocation of blocks for a file—essentially to give the filesystem more information about the programmer's plans for the file so that better allocation decisions can be made. Over time, additional features have been added to `fallocate()`, including the ability to punch holes in or to zero-out ranges of a file. 

There are quite a few similarities between `FALLOC_FL_INSERT_RANGE` and `FALLOC_FL_COLLAPSE_RANGE`. Both must be the only flag passed to `fallocate()` (other options allow ORing in multiple flags), require that the offset and length specified are multiples of the filesystem's logical block size, and both are only implemented for the XFS and extent-based ext4 filesystems. Also, they are restricted to working within the existing file, so the range covered by offset + length must not stretch beyond the current end of file (EOF). 

For inserting a range, the basic algorithm is the same for both XFS and ext4. Once the offset and length parameters are validated (i.e. block-aligned and not past EOF), the file size is increased by the length. The extent containing the logical block number for offset is then examined to see if that block number is the first in the extent. If not, the extent is split so that it starts with the block number corresponding to offset. Then, starting with that extent, all extents from there to the EOF are shifted over (i.e. to the right) by the length, which leaves behind a hole located at the offset with the specified length. 

Once that is done, callers can fill that hole by writing whatever data they want into it—hopefully not just ads. Reading from that region before writing to it will return zeroes, as with other holes punched in files. 

Beyond the changes to the kernel filesystem layer (which are minimal), XFS, and ext4 (which are more extensive), Jeon has also added a number of test cases to xfstests. There are simple tests of the insert range feature, as well as more complicated tests that do multiple inserts or inserts coupled with collapse operations to try to stress both of these features. In addition, he has added support for an "finsert" command to the `xfs_io` program from [xfsprogs](<http://oss.sgi.com/cgi-bin/gitweb.cgi?p=xfs/cmds/xfsprogs.git;a=summary>). 

Jeon's patch set is up to version 8 at this point; there have been lots of suggestions for changes along the way, but little in the way of fundamental opposition. Given that the collapse range capability was added, it would seem likely that insert range will follow along before too long.
