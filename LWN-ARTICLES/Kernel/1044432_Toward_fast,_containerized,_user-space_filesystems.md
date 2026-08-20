---
title: "Toward fast, containerized, user-space filesystems"
url: https://lwn.net/Articles/1044432/
date: "November 6, 2025"
category: "Filesystems-In user space"
author: "By Jonathan Corbet November 6, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 6, 2025

Filesystems are complex and performance-sensitive beasts. They can also present security concerns. Microkernel-based systems have long pushed filesystems into separate processes in order to contain any vulnerabilities that may be found there. Linux can do the same with the [Filesystem in Userspace](<https://docs.kernel.org/filesystems/fuse/>) (FUSE) subsystem, but using FUSE brings a significant performance penalty. Darrick Wong is working on ways to eliminate that penalty, and he has [a massive patch set](<https://lwn.net/ml/all/20251029002755.GK6174@frogsfrogsfrogs>) showing how ext4 filesystems can be safely implemented in user space by unprivileged processes with good performance. This work has the potential to radically change how filesystems are managed on Linux systems. 

One of the biggest challenges faced by a filesystem implementation is the need to parse and maintain a complex data structure that can be entirely under an attacker's control. It is possible to compromise almost any Linux filesystem implementation with a maliciously crafted filesystem image. While most filesystems are robust against corrupted images in general, _malicious_ corruption is another story, and most filesystem developers only try so hard to protect against that case. For this reason, mounting a filesystem, which would otherwise be an inherently safe thing to do when certain protections (against overmounting system directories or adding setuid binaries, for example) are in place, remains reserved to privileged users. 

If the management of filesystem metadata is moved to user space, though, the potential for mayhem from a malicious image is greatly reduced. FUSE allows exactly that, but the overhead of passing all filesystem I/O over the connection to the user-space FUSE server makes FUSE filesystems slow. Wong's attempt to address this problem is somewhat intimidating at first look; it is a collection of five independent patch topics, most of which have multiple sub-parts. It comprises 182 patches in total. There is a lot of complexity, but the core idea is relatively simple. 

#### Iomap

Filesystems move a lot of data around. Much of the added cost of a FUSE filesystem comes from the need to pass data between the kernel and the FUSE server as filesystem operations are requested. If a process writes some data to a file on a FUSE filesystem, the kernel must pass that data to the user-space FUSE server to implement that write; the server will, almost certainly, then pass the data _back_ to the kernel to actually land it in persistent storage. An obvious way to improve this situation would be to somehow keep the data movement within the kernel, and just have the FUSE server tell the kernel where blocks of data should be read from or written to. That would allow the FUSE server to handle the metadata management while removing the extra cost from the I/O path. 

In the end, much of a filesystem's job consists of maintaining mappings between logical offsets within files and physical locations on persistent storage. Once that is done, file I/O boils down to using those mappings to move blocks of data back and forth — a task that is independent of any given filesystem. (Of course, every filesystem developer reading this text is now seething at this extreme oversimplification; there is little to be done for that.) The kernel has offered various mechanisms for managing this mapping, including [buffer heads](<https://lwn.net/Articles/930173/>), which were part of the first public release of the Linux kernel. 

In more recent times, though, this mapping task is supported in the kernel by the [iomap layer](<https://docs.kernel.org/filesystems/iomap/>). It was first [introduced](<https://git.kernel.org/linus/ae259a9c8593>) by Christoph Hellwig (based on older code from the XFS filesystem) for the 4.8 release in 2016, and other filesystems have been slowly making use of it since then. The iomap layer abstracts out a lot of details, simplifying matters on the filesystem side. At its core are [two callbacks](<https://docs.kernel.org/filesystems/iomap/design.html#struct-iomap-ops>) that filesystems must provide: 

```
int (*iomap_begin)(struct inode *inode, loff_t pos, loff_t length,
        		       unsigned flags, struct iomap *iomap,
    		       struct iomap *srcmap);
        int (*iomap_end)(struct inode *inode, loff_t pos, loff_t length,
    		     ssize_t written, unsigned flags, struct iomap *iomap);
```

Without getting into the details, `iomap_begin()` requests the filesystem to specify the on-disk mapping for the given `inode` over the range of `length` bytes starting at `pos`. When the kernel is done with the mapping, it will inform the filesystem with a call to `iomap_end()`. In between, the kernel may well use that mapping to move data between memory and the filesystem's storage device. 

[Sub-Part 4](<https://lwn.net/ml/all/176169810144.1424854.11439355400009006946.stgit@frogsfrogsfrogs>) of Wong's series adds two new operations, `FUSE_IOMAP_BEGIN` and `FUSE_IOMAP_END` to the FUSE API. These operations correspond to the two callbacks above, allowing a user-space filesystem to build an I/O mapping in the kernel, which can then use that mapping to perform many I/O operations directly, without having to involve the user-space server further. While the longer-term goal is to enable unprivileged filesystem mounts, the ability to use iomap in FUSE is restricted to processes that have the `CAP_SYS_RAWIO` capability. 

Providing basic iomap access can speed FUSE servers by avoiding the need to move file data between the kernel and user space, but there is more to be done to reach a high level of performance. One step is in [this series](<https://lwn.net/ml/all/176169812012.1426649.16037866918992398523.stgit@frogsfrogsfrogs>): it allows the kernel to cache iomap mappings created by the FUSE server. That reduces the number of round trips to the server required, but it is also needed to correctly manage mappings in cases where I/O might cause them to change. Another performance improvement comes with [this series](<https://lwn.net/ml/all/176169811533.1426244.7175103913810588669.stgit@frogsfrogsfrogs>), which moves much of the management of timestamps and access-control lists into the kernel. 

Finally, [this short series](<https://lwn.net/ml/all/176169812502.1427080.11949246505492038165.stgit@frogsfrogsfrogs>) allows a privileged mount helper to set a special bit enabling the FUSE server process to use the iomap capability, regardless of whether it has `CAP_SYS_RAWIO`. That makes it possible for a server process to run in an unprivileged mode, opening up the possibility of implementing filesystems in unprivileged processes that are unable to compromise the system. 

#### User space

That, however, is only the kernel side of the equation. There are another five sub-parts of the series that add the equivalent support to the [libfuse](<https://github.com/libfuse/libfuse?tab=readme-ov-file#libfuse>) user-space library. Yet another six sub-parts add support for the new FUSE features to [fuse2fs](<https://www.man7.org/linux/man-pages/man1/fuse2fs.1.html>), the server program that implements the ext4 filesystem (and ext3 and ext2 as well) in user space. As Wong points out in [sub-part 1](<https://lwn.net/ml/all/176169817482.1429568.16747148621305977151.stgit@frogsfrogsfrogs>), the results are encouraging: 

> The performance of this new data path is quite stunning: on a warm system, streaming reads and writes through the pagecache go from 60-90MB/s to 2-2.5GB/s. Direct IO reads and writes improve from the same baseline to 2.5-8GB/s. FIEMAP and SEEK_DATA/SEEK_HOLE now work too. The kernel ext4 driver can manage about 1.6GB/s for pagecache IO and about 2.6-8.5GB/s [for direct I/O], which means that fuse2fs is about as fast as the kernel for streaming file IO. 

He does also acknowledge that the results for random buffered I/O are not as good at this point. 

The patch series includes a fair amount of support for running unprivileged FUSE filesystem servers, further containing any fallout from a compromised (or malicious) FUSE server. The whole series ends with [a 33-patch sub-part](<https://lwn.net/ml/all/176169819804.1433624.11241650941850700038.stgit@frogsfrogsfrogs>) adding testing support for ext4 under FUSE. 

#### Prospects

This is a lot of work that offers some obvious benefits, but it is also a lot for the filesystem developers to absorb. Even so, Wong said in the cover letter that he would like to merge these patches for the 6.19 kernel release. That seems rather ambitious. Hellwig [asked](<https://lwn.net/ml/all/aQHf3UGaURFzC17U@infradead.org>) for the series to be split up and made easier to review; it is not clear whether Wong intends to do that. He has not gotten around to documenting the iomap changes, though that work must surely be at the top of his to-do list. And, of course, all of this work will need to be reviewed, and likely revised, before it can be merged. 

So, in summary, it would be somewhat surprising to see these changes actually land for 6.19. But, given the obvious value that this work brings, Wong may well succeed in upstreaming it in the not-too-distant future. If his results bear out in wider usage, distributors and system integrators could start shipping systems with FUSE-implemented filesystems, which would be a significant change from how Linux systems have worked since the beginning. Linux may never be a microkernel, but it may soon look rather more microkernel-like than it does now.
