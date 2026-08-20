---
title: An introduction to EROFS
url: https://lwn.net/Articles/934047/
date: "June 7, 2023"
category: "Filesystems-EROFS"
author: "By Jake Edge June 7, 2023 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 7, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Gao Xiang gave an overview of the [Extended Read-Only File System](<https://en.wikipedia.org/wiki/EROFS>) (EROFS) in a filesystem session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). [EROFS](<https://docs.kernel.org/filesystems/erofs.html>) was added to Linux 5.4 in 2019 and has been increasingly used in places beyond its roots as a filesystem for Android and embedded devices. Container images based on EROFS are being used in many places these days, for example. 

Unfortunately, this session was quite difficult for me to follow, so the report below is fragmentary and incomplete. There is a [YouTube video](<https://www.youtube.com/watch?v=bQ1GT2U4GnU>) of the session, but it suffers from nearly inaudible audio, though perhaps that will be addressed before long. The [slides](<https://docs.google.com/presentation/d/16LwD-F0IKy8okkC8qB94v2YgDcWBBE26rmMn_s1Mqxc>) from the session are also available. 

EROFS is a block-based, read-only filesystem with a "very simple" format, Xiang began. The earlier read-only filesystems had many limitations, such as not supporting compression, which is part of why EROFS was developed. EROFS stores its data in a block-aligned fashion, which is page-cache friendly; that alignment also allows direct I/O and [DAX](<https://www.kernel.org/doc/html/latest/filesystems/dax.html>) filesystem access. 

[ ![\[Gao Xiang\]](https://static.lwn.net/images/2023/lsfmb-xiang-sm.png) ](<https://lwn.net/Articles/934087/>)

[SquashFS](<https://docs.kernel.org/filesystems/squashfs.html>) is another read-only filesystem, but it does not store its compressed data in a block-aligned fashion, which increases the I/O overhead. EROFS does its compression into fixed 4KB blocks in the filesystem, while SquashFS uses fixed-sized blocks of uncompressed data. In addition, SquashFS does not allow random-access in its directories, unlike EROFS; that means SquashFS requires linear searches for directory entries. 

Replacing tar or cpio archives with a filesystem is a potential use case for EROFS. There has been a proposal from the confidential-computing community for a kernel tarfs filesystem, which would allow guest VMs to efficiently mount a tar file directly. But EROFS would be a better choice, he said. There is a proof-of-concept patch set that allows directly mounting a downloaded tar file using EROFS that performs better than unpacking the tarball to ext4, then mounting it in the guest using [overlayfs](<https://docs.kernel.org/filesystems/overlayfs.html>). 

There are still problems with this approach, including a lack of sharing in the page cache between guests that are using the same tar archive. Aleksa Sarai agreed that there was a problem with that, but thought that eliminating tar archives as the underlying format would go a long toward fixing it—along with a bunch of other problems. He also said that the EROFS approach is better than what's being done today, but believes that replacing the tar format in container images is needed. 

There is currently a lot of effort that goes into optimizing image layout that is all needed solely due to the tar format; "in my mind, this is insanity", Sarai said. The community needs to stop expending so much energy working around the limitations of the tar format. There may be 500 instances of Bash in the guests on a system, but they cannot share the same inode in a tar-based format, so they are treated as distinct files. But the tar format is going to continue to need to be supported, Xiang said, so a compatible solution is needed. 

He continued with features of EROFS, including the ability to do chunk-based deduplication of file data. The typical [use case](<https://www.alibabacloud.com/blog/faster-container-image-loading-speed-with-nydus-rafs-and-erofs_599012>) is for systems using EROFS with [Nydus](<https://github.com/dragonflyoss/image-service/blob/master/docs/nydus-design.md>). EROFS optionally supports per-file compression with LZ4/LZMA, but uses smaller compression block sizes, which reduces the memory amplification that occurs with SquashFS. The data is decompressed in-place in order to reduce extra copies. 

Recent use cases for EROFS take three basic forms. The first is an EROFS full image; those are used in compressed form for space saving at the cost of some performance, or uncompressed and shared among guests with DAX or [FS-Cache](<https://docs.kernel.org/filesystems/caching/fscache.html>). The second is to have an EROFS metadata-only image with an external source for the file data, such as a tar archive or other binary format. The third is to use EROFS with overlayfs as described in the [previous session on composefs](<https://lwn.net/Articles/933616/>). 

Using EROFS could potentially increase performance for machine-learning data sets, Gao said. These data sets often have millions of small files in a single directory; the training process will read the entire directory and choose files randomly from the list. Because of its compact layout, EROFS is potentially twice as fast as ext4 for those kinds of operations. 

The session wound down with some discussion about using the [clone-file-range `ioctl()` operation](<https://man7.org/linux/man-pages/man2/ioctl_ficlonerange.2.html>) to do an overlayfs [copy_up](<https://docs.kernel.org/filesystems/overlayfs.html#non-directories>) on files. A copy_up is performed when the lower-layer file is accessed for write; the file gets copied to the upper layer before it can be modified. If the layers are loopback-mounted files from the same filesystem, a copy-on-write operation could be done instead. Amir Goldstein seemed to think that something like that is possible and would be useful, but there is work needed to get there.
