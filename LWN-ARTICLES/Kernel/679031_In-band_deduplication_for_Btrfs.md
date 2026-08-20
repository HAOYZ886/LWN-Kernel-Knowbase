---
title: "In-band deduplication for Btrfs"
url: https://lwn.net/Articles/679031/
date: "March 9, 2016"
category: "Btrfs; Filesystems-Btrfs"
author: "March 9, 2016 This article was contributed by Neil Brown"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

March 9, 2016

This article was contributed by Neil Brown

"In-band deduplication" is the process of detecting and unifying duplicate data blocks as files are being written, rather than at some later time. Btrfs support for this feature has been under development since at least [early 2013](<https://lwn.net/Articles/550545/>). Quite recently it reached the point where developer Qu Wenruo thought it was sufficiently ready to send Btrfs maintainer Chris Mason a [pull request](<http://thread.gmane.org/1455774178-3595-1-git-send-email-quwenruo@cn.fujitsu.com>) (as yet unanswered) hoping that it might be added to the kernel during the 4.6 merge window. While this is far from a magic bullet that will suddenly remove all the waste in your filesystem that is caused by duplicate data, there are use cases where it could bring real benefits.

#### Offline and in-band

It has been possible to unify duplicated blocks in Btrfs since Linux 3.12 when the `BTRFS_IOC_FILE_EXTENT_SAME` `ioctl()` command was [added](<http://git.kernel.org/torvalds/c/416161db9b63>) (it has since been [renamed `FIDEDUPERANGE`](<http://git.kernel.org/torvalds/c/54dbc1517237>) when the basic `ioctl()` handling was moved to the VFS, so it could someday be used by other filesystems). This `ioctl()` is given a range of bytes in the file (which must be aligned to the filesystem block size) and a list of offsets in other files (or possibly the same file). This identifies two or more ranges of the same size that are claimed to be identical. Btrfs reads all the blocks, checks that they are in fact identical, and then changes its internal bookkeeping so that all files point to one set of blocks on the storage medium. The space used by the other copies of the data will typically become free space available for reallocation.

To understand how this works it is necessary to understand how Btrfs makes use of "[extents](<https://en.wikipedia.org/wiki/Btrfs#Extents>)". An extent is a contiguous set of blocks on the storage device and also a contiguous set of blocks in a file (at least initially). It has a "logical address" that is mapped to a physical location on one, or perhaps more, of the underlying devices. An extent can become part of several different files, either through snapshots, reflinks, or deduplication. To allow this storage space to ultimately be reused, Btrfs maintains a reference count for each extent and will not re-use any of the space until that count becomes zero. 

It is not generally possible to split an extent. If a single block is written into the middle of a large extent, the "copy on write" design of Btrfs requires that either the whole extent be copied or that the file's indexing information be changed to point to the first half of the extent, then the new block, then the remainder of the extent. Btrfs takes the latter approach. 

When `FIDEDUPERANGE` is used, the file indexes will be updated to point to the relevant extents or partial extents of the source file, and the reference counts on the various extents will be increased or decreased as appropriate. This may not release quite as much space as expected since there may have been extents that were only partially identical between two files. The identical part in one extent may have no files including it any more, but the space will not be freed until the whole extent is no longer referenced. 

[`duperemove`](<https://github.com/markfasheh/duperemove>) is a tool that can be used to examine a set of files, look for duplicated regions, and call the ioctl to remove that duplication from the underlying storage. This can be useful anywhere that files are likely to be completely or largely the same, but where they need to be kept separate for administrative or other practical reasons. A common use case is filesystem images used for virtualization or the multiple similar-but-not-identical file sets used by containers. Running `duperemove` from time to time could save a lot of disk space.

This functionality was referred to in the initial commit as "offline deduplication", which is a little confusing since "offline" in the context of filesystems usually implies that the filesystem isn't mounted, and that is not the case here. It is more like "on-demand" deduplication and is [sometimes](<https://btrfs.wiki.kernel.org/index.php/Deduplication>) referred to as "batch" deduplication. This contrasts with the new work that could be called "time-of-write" or transparent deduplication, but is called "in-band" deduplication in this patch set.

When `duperemove` runs, it computes a hash value for every block in every file and then compares these, looking for ranges of matching blocks. When it finds suitable ranges, it requests the deduplication. In-band deduplication uses the same idea of hashing blocks of data, but in many other respects it is quite different.

In-band duplication works in units of extents of a specific size — a deduplication block size of up to 8MB can be configured; the default is 128KB. When data is written out to storage, blocks are gathered into extents of this size whenever possible. If there are only sufficient blocks being written for a smaller extent, that extent will be written without deduplication. For every extent of the target size, the hash (currently SHA-256, though possibly configurable in the future) is calculated. If another extent can be found with the same hash, then a new reference is added to that extent and the new data is not written. If no match is found, the data is written as normal and the hash is stored together with the logical address of the extent so that it is available for future matching. 

This process can be enabled for a whole Btrfs filesystem using the "`btrfs dedup enable`" command and then disabled for individual files or directories by using the "`btrfs property set`" command to set the "`dedup`" property to "`disable`". Setting this on a directory causes all files or directories created within that directory to inherit the setting. This means that enabling deduplication on only a subset of a filesystem is possible, but a little bit clumsy. 

#### Two back-ends

There are two separate back-ends for storing the mapping between hashes and extents; the fact that the implementation of these back-ends is kept cleanly separate from their use is rightfully highlighted as a strength of this patch set.

One of the two back-ends is an in-memory mapping. Two [red-black trees](<https://lwn.net/Articles/184495/>) are created when the filesystem is mounted; whenever new data is written to a file on which deduplication is enabled, new entries are added to the trees. One tree maps from hash to logical address and is used to see if a suitable extent exists that already stores the required data. The second provides a reverse mapping that allows hash entries to be deleted when an extent is freed after its reference count reaches zero. The size of these trees is limited by a configurable entry count that defaults to 32768. It would not be surprising to see that replaced or augmented with a "[shrinker](<https://lwn.net/Articles/550463/>)" callback from the memory management subsystem, so that it could shrink only when memory is tight.

The second mechanism stores these mappings in the filesystem. Btrfs has flexible data structures for storing all sorts of metadata and data on disk using a number of distinct B-trees, one for each [subvolume](<https://lwn.net/Articles/579009/>), one for the extent reference counts, one for storing checksums, etc. The in-band deduplication patchset adds another B-tree that has two types of keys: one for lookup by hash and another for lookup by extent address.

Looking up the hash in the B-tree is not quite so straightforward as one might hope, since Btrfs has a fixed format for the lookup key: a 64-bit object-ID, an eight-bit type, and a 64-bit offset. It is not possible to store the full 256-bit hash in the key and even storing 128 bits in the two 64-bit fields would be problematic. Btrfs requires that all keys be unique, so that approach would effectively limit the hash size to 128 bits.

The approach chosen is to store 64 bits of the hash in the object-ID, and the logical address of the extent in the offset field. Each key is accompanied by a variable-length data field and this is used to store the full hash. If the hashes of two extents collide in those 64 bits, the offsets will still be different, so the keys will be unique. 

To handle these collisions a lookup first performs a regular B-tree search for a key with the appropriate hash bits in the object-ID and `U64_MAX` in the offset field. This will provide the last possible location where the target hash could be stored. A linear search is then performed searching backward and comparing the full hash until a match is found or there are no more keys with the required hash fragment. 

Unlike with in-memory lookup there is no mechanism to limit the number of hash entries stored, beyond normal filesystem-full checks that might prevent a new extent from being written.

One difference between this in-band deduplication and the on-demand approach that is worth highlighting is the dependence placed on the hash. `duperemove` uses a hash only to guide the search for duplicate blocks — the hash is not authoritative and Btrfs will not allow the deduplication to happen if the identified regions are not byte-for-byte identical. In-band duplication as currently implemented does not perform that comparison. If the hash matches, then the extents are assumed to match. This means the correctness of the deduplication is completely dependent on the uniqueness of the hash, so the extra effort to make use of all 256 bits is easy to justify. Whether that complete dependence is itself justified is not an easy question to answer. It is certainly _extremely_ unlikely for two distinct blocks to have the same hash, but it is also certainly quite possible. It would only need to cause corruption once to be extremely embarrassing. Adding byte-for-byte comparison is [on the planned feature list](<http://mid.gmane.org/56D8E42C.6030106@cn.fujitsu.com>) but, according to Wenruo, "not anytime soon". 

In-band deduplication brings the benefit of being automatic, but has a cost that it will probably miss duplication that `duperemove` could find. Different alignment of extents between files would completely defeat the duplicate detection, as would creating files before deduplication was enabled. The former could be improved to some degree with a smaller extent size, though that would brings costs of its own. As is so often the case, finding the most effective solution — which could include a mix of offline and in-band deduplication — will be highly dependent on each particular use case.

#### Test, test, and test

Wenruo assures us that this patch set has seen quite a lot of testing and that it has been some time since the last of the bugs found by that testing was fixed, which is encouraging. Some [new tests](<http://thread.gmane.org/1456301196-15874-1-git-send-email-quwenruo@cn.fujitsu.com>) have been submitted for the [xfstests](<http://oss.sgi.com/cgi-bin/gitweb.cgi?p=xfs/cmds/xfstests.git;a=summary>) test suite specifically to exercise the deduplication and to check for some of the bugs that have been found and fixed.

One aspect of testing that seemed strangely absent from the pull request was any hint of how in-band deduplication affects performance. It is to be expected that this sort of functionality would slow writes down, particularly when the table of hashes is not kept in memory. It could also speed up some writes if lots of duplicate extents are found. Some indication of the sort of performance change experienced would certainly help to complete the picture.

But maybe the hope is to crowdsource that testing. There are so many different possible hardware configurations and usage scenarios to test that it is hard for one developer to even begin to give meaningful results. A large community, on the other hand, can try lots of things in parallel. As Wenruo noted in his pull request, there is still work to be done, but it should be quite ready for people to test.

Trying it out requires some [patches](<http://thread.gmane.org/1455776300-11234-1-git-send-email-quwenruo@cn.fujitsu.com>) to the `btrfs-progs` along with the Git tree from the pull request, but that should be no challenge for those who enjoy compiling their own kernels. I'm sure additional results would be most welcome.
