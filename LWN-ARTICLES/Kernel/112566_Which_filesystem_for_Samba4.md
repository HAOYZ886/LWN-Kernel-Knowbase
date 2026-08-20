---
title: "Which filesystem for Samba4?"
url: https://lwn.net/Articles/112566/
date: "November 23, 2004"
category: "Filesystems-Extended attributes; Filesystems-ext3; Samba"
author: 
---

Andrew Tridgell has been hacking away on Samba 4 for a while now; that project has gotten to the point that he has [started doing some performance testing](<https://lwn.net/Articles/112567/>). His first set of results looked like this (numbers in MB/sec): 

> Filesystem| No xattr| With xattr  
> ---|---|---  
> ext2| 68| 64  
> ext3| 67| 58  
> xfs| 62| 40  
> xfs 2K inode| 63| 58  
> tmpfs| 69| \--  
> jfs| 36| 29  
> reiser3| 58| 44  
  
These results show that all filesystems slow down when extended attributes are used. This matters for Samba 4 because Windows filesystems make heavy use of extended attributes. As Tridge put it: 

The high cost of xattr support is a bit of a problem.... I hope we can reduce the cost of xattrs as otherwise Samba4 is going to be seriously disadvantaged when full windows compatibility is needed. I'm guessing that nearly all Samba installs will be using xattrs by this time next year, as we can't do basic security features like WinXP security zones without them, so making them perform well will be important. 

The cause of the performance problems is not particularly mysterious. Most filesystems store extended attributes in a special data block, away from the rest of the associated file's metadata. So working with a file's extended attributes forces the filesystem to go out and read another block from the drive. The extra transfers and seeks take their toll on performance, as can be seen in the numbers above. 

A pointer to the solution can be seen there as well. The "xfs 2K inode" results were obtained by turning on the XFS large inode option. This option expands the size of the on-disk inode structure, making room for the extended attributes to be stored there. When the inode is read from the drive, the extended attributes come with it, and no separate I/O is required to work with them. When this option is enabled, the performance hit for using extended attributes with XFS is much reduced. 

It turns out that [a large inode patch for ext3](<https://lwn.net/Articles/112569/>) has been in the works for a while; it has passed muster with the ext3 developers, but has not yet been pushed into the mainline. Tridge [tried this patch](<https://lwn.net/Articles/112571/>) and was pleased with the results: 

Using a 256 byte inode on ext3 gained a factor of up to 7x in performance, and only lost a very small amount when xattrs were not used. It took ext3 from a very mediocre performance to being the clear winner among current Linux journaled filesystems for performance when xattrs are used. Eventually I think that larger inodes should become the default. 

First, however, the patch must be merged. With testimonials like this, that merger is likely to happen in the relatively near future. 

One interesting mystery remains, however: Tridge [gets notably better results](<https://lwn.net/Articles/112572/>) with 2.6.10-rc2-mm2 than what he gets with 2.6.10-rc2. As of this writing, nobody seems to have an explanation for why ext3 should perform that much better in the -mm kernel. Inquiring minds very much want to know, however, and Andrew Morton is working at finding out which patch makes the difference.
