---
title: Blurred boundaries in the storage stack
url: https://lwn.net/Articles/680708/
date: "March 24, 2016"
category: fallocate
author: "March 24, 2016 This article was contributed by Neil Brown"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

March 24, 2016

This article was contributed by Neil Brown

It has been [said](<https://lwn.net/Articles/571995/>) that an important part of a maintainer's role is to say "no". Just how this "no" is said can define the style and effectiveness of a maintainer. Linus Torvalds recently displayed just how effective his style can be when saying "[no](<http://mid.gmane.org/CA+55aFx_NPn0VYk=+Ad5S_r=D6J1xFmWmf7JzQ7RmkwKmdkYOg@mail.gmail.com>)" to a pair of fairly innocuous [patches](<http://thread.gmane.org/20160302040932.16685.62789.stgit@birch.djwong.org>) to add a new `ioctl()` command for block devices — patches in their fifth revision that had already received "Reviewed-by" tags from Christoph Hellwig:

NAK, just based on annoyance with the randomness of this interface 

It became clear that Torvalds only had a fairly general understanding of the underlying functionality and didn't much care about it anyway. What he cared about, as he said, was the interface. It seemed both ""too specific"" and too generic; ""too 'future-proofing'"".

These complaints led to a wide-ranging discussion that brought out a number of underlying issues, drew parallels between disparate parts of the storage stack, and resulted in a new interface proposal that gives quite a different flavor to the same basic operations.

#### The heart of the matter

Modern storage devices can do a lot more with stored data than simply read or write arbitrary blocks. Of the other operations the best known is doubtlessly "discard". This operation, named `TRIM` in the ATA protocol and `UNMAP` in SCSI, tells the storage device that the data in some data blocks is no longer needed. It is well-known because it is both valuable and problematic. Some SSDs work better if unused data is regularly trimmed, but trim implementations work differently on different devices, both in terms of efficiency and effectiveness. This variation means that users often need to know precise details of their hardware to achieve the best performance.

There is an operation that is the inverse of discard that is important for thin-provisioned devices. Thin provisioning allows a storage array to appear to be extremely large, while only having physical capacity for a much smaller amount of storage. As data is written, the available storage is allocated to the target addresses. As the free space shrinks, the device administrator is alerted and action can be taken, which could include acquiring extra physical capacity.

A particularly useful operation when using a thin-provisioned device is to request that storage space be allocated before actually writing data to it. This makes it possible to report allocation problems earlier and to avoid unpleasant surprises. The SCSI spec refers to these unwritten allocations as `ANCHORED` blocks, and supports anchoring with the `WRITE SAME` SCSI command, which writes a particular block of data (often zeros) to multiple locations over a given range of addresses.

The Linux block layer has an interface, `blkdev_issue_zeroout()`, that combines both the de-allocation of discard and the pre-allocation of `WRITE SAME` with the more generic goal of zeroing out a range of blocks on a device. Depending on the capabilities of the device and on the "discard" flag that is passed to the function as a hint, it will issue a discard request (i.e. `TRIM` or `UNMAP`), a `WRITE SAME` request, or write a zeroed page of memory to every block in the range. Future reads are guaranteed to return zeros, but pre-allocation or de-allocation happens on a best-effort basis. 

The "discard" hint flag and the possible issuing of a discard request is a relatively recent addition and is, importantly, different from the similar `blkdev_issue_discard()` interface. The latter will issue a discard even if the result might be that subsequent reads return random data. `blkdev_issue_zeroout()` will only issue a discard if future reads will reliably return zeros. 

#### Simple patches for a simple problem

The pair of patches that Darrick Wong posted does two things. Primarily they add a new `ioctl()` command so that the "discard" flag can be set from user space; the existing `BLKZEROOUT` `ioctl()` calls `blkdev_issue_zeroout()` but always sets the "discard" flag to zero. Hoping not to have to create yet-another-command if even more functionality is ever added to `blkdev_issue_zeroout()`, Wong defined the new `BLKZEROOUT2` with room for expansion: 32 flags of which only one was used, and even some "padding" fields that must be zero now, but could be defined later.

The other effect of these patches is to purge parts of the page cache for the block device when blocks are zeroed. Normal reads and writes on a block device (e.g. `/dev/sda`) are cached in the page cache. An `O_DIRECT` write is instead sent directly to the device, which could make it inconsistent with the page cache. To avoid such inconsistency, the corresponding pages of the page cache are removed when an `O_DIRECT` write happens. `BLKZEROOUT` is much like an `O_DIRECT` write, so, with the patches applied, both it and `BLKZEROOUT2` will purge the page cache.

Torvalds's response seems to be based on an intuitive "it doesn't feel right" rather than clear logical reasoning. One flaw he identified was not actually present in the code; it boiled down to "I absolutely _detest_ code that tries to be overly forward-thinking", which is a little surprising given the [problems](<https://lwn.net/Articles/585415/>) there have been with system calls not having a suitable `flags` argument. Most of the rest is summed up by his comment: ""So the whole patch looks pointless."" He did approve of purging the page cache, though.

As the discussion progressed and requirements were more explicitly stated, the source of Torvalds's discomfort became clearer. The operations of interest deserved to be thought about at a much higher level than just `ioctl()` commands for a block device. They are much more like operations on a file — to allocate and de-allocate backing store.

The Linux `fallocate()` system call has a flag `FALLOC_FL_PUNCH_HOLE`, which is a lot like `TRIM`, particularly the style of `TRIM` that causes future reads to return zeros. `fallocate()` also has that `FALLOC_FL_ZERO_RANGE` flag, which is a good match for `WRITE SAME` or writing zeros. Rather than providing an `ioctl()` command that seems focused on matching low-level functionality provided by certain hardware, using `fallocate()` would reuse an existing high-level interface that is described in terms of the needs of applications. Existing `fallocate()` implementations already purge the page cache as appropriate, so had this approach been used instead of the initial `BLKZEROOUT` `ioctl()` command, it is likely that those implementations would have been used as a guide, so we would not have the current situation where zeros can be written without any purge.

Wong provided a [new patch set](<http://thread.gmane.org/20160305005556.29738.66782.stgit@birch.djwong.org>) that added `fallocate()` support for block devices; this received much warmer support from Torvalds. He found a few little nits, but admitted that ""on the whole, I like it"". This was a fitting close to a maintainership interaction done really well: Torvalds followed his intuition and complained about things that bothered him, despite not having a full picture of the problem space. Wong responded directly, called Torvalds out where he was clearly wrong, and attempted to justify other choices with extra details. A more complete picture was formed, against which preferences could be explained more coherently. Finally a resolution was found, implemented, and approved — apparently to everyone's satisfaction. This is a model worth following.

#### An enlightening tangent

While the conclusion to the main thread of discussion was that treating block devices a bit more like files could make it easier to work with new hardware, there was a [sub-thread](<http://thread.gmane.org/20160303223952.GE24012@thunk.org>) that seemed to head in a complementary direction.

There appear to be a number of user-space file servers — Ceph was given as an example — that use a local filesystem to store data, but aren't really interested in many of the traditional semantics of a filesystem. A good example of this is the `O_NOMTIME` flag that was [discussed](<https://lwn.net/Articles/643892/>) last year. These file servers really just want space to store data and want reads and writes to that space to be passed down to the device with minimal friction from the filesystem.

In much the same way as described earlier for thin provisioning, these file servers need to be able to allocate space and write to it later. While they wouldn't object to that space being filled with zeros, they really don't care about the contents of the space, but they do care about the allocation and subsequent writes being fast.

Filesystems do support pre-allocating space with `fallocate()`, but they typically do so by recording which blocks have been written and which have only been anchored. This means that each subsequent write needs to spend time updating metadata: extra work that brings no value to the file server.

At the beginning of the sub-thread, Ted Ts'o mentioned in passing that he had out-of-tree patches that provide a flag, `FALLOC_FL_NO_HIDE_STALE`, that would do exactly what the file servers want: allocate space so that future writes happen with no further metadata updates. In general, this can be a security issue since reading data from those ranges could return potentially sensitive data belonging to some other user.

Ts'o's patches restrict this operation to a single privileged group ID. There were suggestions that a mount option should be used instead of, or maybe as well as, a special group ID. There were also observations that using the flag in containers could lead to unexpected information leaks. Possibly the most vocal critic was Dave Chinner who was [blunt](<http://mid.gmane.org/20160303231050.GU29057@dastard>): ""it is dangerous and compromises system security. As such, it does not belong in upstream kernels."" An example he gave of possible information leaks was automated backups. While the application that pre-allocated space may be trusted to never look at the stale data, once it leaks out in backups it seems to be more exposed.

Torvalds [wasn't convinced](<http://mid.gmane.org/CA+55aFwHLJffmN-Dw=yZCGKzxe_2Tm9h2GjdaFL3JdvYXNstRw@mail.gmail.com>) by Chinner's fears; his only requirement is that it isn't _too_ easy to do something dangerous. He has always been in favor of providing functionality if people are actually going to use it, so the fact that Ts'o has this out-of-tree patch that is widely used within Google does carry weight. It was also noted that the presence of these performance issues has [already caused](<http://mid.gmane.org/CAC6JEv-CGaphHj6hVXUhFS+Dc4jS46nO1yOfKF5yA8AnDwqgOw@mail.gmail.com>) Ceph developers to give up on using a local filesystem and to instead start using block devices directly, so the issues are clearly real. If performance benefits can be clearly demonstrated and application developers affirm that they would use the functionality, then remaining barriers are unlikely to stand for long. 

If we step back for a moment to grasp the big picture, what we see here is the cluster filesystem using a local filesystem a lot like a logical volume manager. It wants storage space of arbitrary size with the ability to expand later. It doesn't care about any metadata except the size, and doesn't care about the initial contents, which in practice could be stale data. This sounds exactly like the logical volumes that LVM2 can provide, though by being embedded in a filesystem they are much easier to manage than LVM2 volumes. In a mirror image of the decision to treat block devices more like files so as to meet the needs of low-level hardware, it seems that we might want to treat files more like block devices so as to meet the needs of high-level filesystems.

As Chinner himself noted, there are synergies here with the "splitting filesystems in two" idea that he [floated](<https://lwn.net/Articles/592091/#splitting>) at the Linux Storage, Filesystem, and Memory Management Summit in 2014. While nothing appears to have come of that yet, it is valuable food for thought and something may yet arise as needs and options become clearer. The distinction that Chinner made between "names" and "storage" certainly seems stronger than the distinction between "files" and "block devices", which is showing its weakness. If the old lines are going to blur, it might be useful to have new lines to focus our thoughts on a clearer overall picture. That way, we might not need to depend so much on the intuition of experienced maintainers.
