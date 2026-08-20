---
title: ext3 metaclustering
url: https://lwn.net/Articles/265285/
date: "January 16, 2008"
category: "Filesystems-ext3"
author: "By Jonathan Corbet January 16, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 16, 2008

The ext3 system uses the classic Unix block pointer method for keeping track of the blocks in each file. For a given file, the on-disk inode structure contains space for twelve block numbers; they point to the first twelve blocks in the file - the first 48KB of space. If the file is larger than that, a 13th pointer contains the address of the first _indirect block_ ; this block contains another 1024 (on a 4K block filesystem) block pointers. Should that not suffice, there's a 14th pointer for the double-indirect block - each entry in that block is the address of an indirect block. And if even that is not enough, there's a 15th entry pointing to a triple-indirect block full of pointers to double-indirect blocks. 

This is a very efficient representation for small files - the kinds of files Unix systems typically held, once upon a time. In current times, when one can forget about that directory full of DVD images and never even notice the lost space, it does not work quite as well - there is a lot of overhead for all of those individual block pointers, and a large data structure to manage. That is why removing a large file on an ext3 filesystem can take a long time - the system has to chase down all of those indirect blocks, which, in turn, forces a lot of disk activity and head seeks. For this reason, contemporary filesystems tend to use extent-based mechanisms to associate blocks with files, but that is not really an option for ext3. 

An additional problem with all those indirect blocks is that filesystem checkers must locate and verify them all. That, again, causes a lot of head seeking and makes fsck run slowly. Slow filesystem checking was the motivation behind [this patch from Abhishek Rai](<http://lwn.net/Articles/264970/>) which attempts to improve performance on filesystems with a lot of indirect blocks. 

The approach taken is relatively simple: the patch just tries to group indirect block allocations together on the disk. The current ext3 code will allocate indirect blocks when they are needed to account for data blocks being added to the file; they are usually placed adjacent to those data blocks. One might think that this placement would speed subsequent accesses to the file, but that is not necessarily so; the reading or writing of the indirect block will tend to happen at a different time than operations on the data blocks. What this placement does accomplish, though, is the distribution of the indirect blocks all over the disk. So a process which must examine all of the indirect blocks associated with a file must cause the disk to do a lot of head seeks. 

The "metaclustering" approach works by reserving a set of contiguous blocks at the end of each block group. Whenever an indirect block is needed, the filesystem tries to get one from this dedicated area first. The end result is that all of the indirect blocks are located next to each other. Should somebody need to read a number of those blocks without being interested in the contents of the data blocks, they can grab them all quickly with minimal seeking. Filesystem checkers, as it happens, need to do exactly that - as does the file removal process. The patch did not come with benchmarks, but the speedup that comes from the elimination of all those seeks should be significant. 

Even so, Andrew Morton [questioned the need](<https://lwn.net/Articles/265286/>) for this patch, worrying that its benefits do not justify the risks that comes with modifying an established, heavily-used filesystem: 

In any decent environment, people will fsck their ext3 filesystems during planned downtime, and the benefit of reducing that downtime from 6 hours/machine to 2 hours/machine is probably fairly small, given that there is no service interruption. 

Others disagreed, though, noting that it's the _unplanned_ filesystem checks which are often the most time-critical. That includes the delightful "maximal mount count" boot-time check which, in your editor's experience, always happens when one is trying to get set up to give a talk somewhere. So this patch might just find eventual acceptance - it should be relatively low-risk and does not require any on-disk format changes. This is a filesystem patch, though, so nobody will be in any hurry to get it into the mainline before a lot of testing and review has been done.
