---
title: Atomic writes for ext4
url: https://lwn.net/Articles/1016879/
date: "April 10, 2025"
category: "Atomic IO operations; Filesystems-ext4"
author: "By Jake Edge April 10, 2025 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
April 10, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Building on the discussion in the two previous sessions on untorn (or atomic) writes, [for buffered I/O](<https://lwn.net/Articles/1016015/>) and [for XFS using direct I/O](<https://lwn.net/Articles/1016406/>), Ojaswin Mujoo remotely led a session on support for the feature on ext4. That took place in the combined storage and filesystem track at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit. Part of the support for the feature is already in the upstream kernel, with more coming. But there are still some challenges that Mujoo wanted to discuss. 

For ext4, a single filesystem block can be written atomically; that [support](<https://lwn.net/ml/all/cover.1730437365.git.ritesh.list%40gmail.com/>) was merged for 6.13, he said. There is [work in progress](<https://lwn.net/ml/all/Z-KWsWHOGJnq8pUp@li-dc0c254c-257c-11b2-a85c-98b6c1322444.ibm.com/>) on doing multi-block atomic writes in ext4. There are two main allocation challenges that need to be handled for multi-block, though: unaligned extents that do not match the hardware alignment requirements and ranges with mixed mappings, for example those that cover both unwritten data and hole sections. 

The ext4 [bigalloc](<https://www.kernel.org/doc/html/latest/filesystems/ext4/bigalloc.html>) feature eliminates the problem with unaligned extents because the cluster size for the filesystem can be set to, say, 16KB, so everything will be aligned on those boundaries. But it is a filesystem-wide setting, even if atomic writes are only needed for a few files, and it requires that any existing filesystem be reformatted to use the feature. Reformatting may not be desirable for all use cases, but multi-block writes with bigalloc is working now. 

Currently, without bigalloc, ext4 does not have a way to guarantee the needed alignment; if an atomic write is done on an unaligned extent, ext4 has no fallback, it simply returns an error to the user. In order to ensure the alignment, Mujoo is [exploring a combination of extsize and forcealign](<https://lwn.net/ml/all/cover.1742800203.git.ojaswin%40linux.ibm.com/>). Extsize is a per-inode alignment "hint" to the allocator that is set with an [`ioctl()`](<https://www.man7.org/linux/man-pages/man2/ioctl.2.html>) command; it will try to allocate all extents to the boundary specified, but can fail. The forcealign extended attribute can be set on a file that has an extsize specified in order to require that allocation alignment; it can be seen as a per-file bigalloc. 

Luis Chamberlain said that he has done some analysis of ext4 using bigalloc with a 16KB cluster size and noticed that some writes are not aligned on 16KB boundaries; he wondered why that was. Ted Ts'o said that bigalloc guarantees that data blocks are aligned to the cluster size, but not metadata blocks, which are still 4KB-aligned. Journal updates, inode updates, and bitmap-allocation-block updates could all cause writes that are not aligned to the cluster size. 

Chamberlain wondered if there was any way to support 16KB writes for ext4 metadata; Ts'o said that it would require ext4 support for [filesystem block sizes larger than the page size](<https://lwn.net/Articles/1009548/>). The buffered I/O path for ext4 would probably need to switch to using iomap, he said; the ext4 developers are interested in getting patches that make that switch and he understands that large-block-size support is fairly straightforward once that happens. 

The idea is to not require any reformatting of the filesystem with extsize and forcealign, Mujoo said. That will require fallbacks for files that are not properly aligned when forcealign is set for them. A "compat" feature flag can be added that can be set on existing filesystems; that will allow older kernels to mount the filesystem. An `ioctl()` command can be added to fix files that are not properly aligned. The forcealign feature might also have use cases outside of atomic writes; for example, it might help with getting properly aligned blocks for use with a [direct-access](<https://docs.kernel.org/filesystems/dax.html>) (DAX) filesystem. 

The problem of mixed mappings affects both bigalloc and non-bigalloc ext4; avoiding atomic writes with mixed mappings should be the goal, but it may not always be met. If a mixed mapping is used for an atomic write, there are three solutions that he sees. The first is to return an error, which might be popular for those who do not want a fallback path. Another is to zero the holes and write them with the rest. Finally, ext4 can do something similar to what XFS is doing: write the data in a new place and atomically change the extent mappings. Ext4 has no infrastructure to support the XFS-like solution, however, so it would add complexity to the solution. 

Mujoo described the roadmap for ext4 atomic-write support. The patch sets for multi-filesystem-block writes using bigalloc and for adding extsize and forcealign support to ext4 are being targeted for Linux 6.16. Subsequent features, including using extsize and forcealign for multi-block atomic writes, exploring an extent-swapping fallback, and enabling buffered atomic writes for ext4, will come later. 

Chamberlain asked if the idea of using the [multi-index feature](<https://docs.kernel.org/core-api/xarray.html#multi-index-entries>) of [XArray](<https://docs.kernel.org/core-api/xarray.html>) had been evaluated as a means to more generically support atomic writes for all filesystems. Mujoo agreed that it would be nice to have VFS support for atomic writes that could be used by more filesystems, but has not really looked at that.
