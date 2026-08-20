---
title: "Merging famfs?"
url: https://lwn.net/Articles/1082687/
date: "July 20, 2026"
category: "DAX; Filesystems-famfs"
author: "By Jake Edge July 20, 2026 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
July 20, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

The [famfs filesystem](<https://github.com/cxl-micron-reskit/famfs#famfs-shared-memory-filesystem-framework---user-space-repo>), which is meant to provide shared access to huge memory-resident files on [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>) and other devices, returned to the [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) (LSFMM+BPF) in 2026. It was [first discussed at LSFMM+BPF 2024](<https://lwn.net/Articles/983105/>) and a [new implementation was described at the 2025 gathering](<https://lwn.net/Articles/1020170/>), but it still has not made its way into the kernel; LWN [looked at a discussion about merging famfs](<https://lwn.net/Articles/1068686/>) back in April 2026. 

John Groves began the session by noting that he is the creator and maintainer of famfs, which is the ""fabric-attached memory filesystem"". He works for Micron, which makes devices with large pools of memory that are accessible to multiple servers; those products are coming soon, though early access to them is available to some. There are some ""interesting management challenges"" for such devices and famfs is meant to allow Linux users to handle them. Since last year's summit, he has become the co-chair of the software and systems working group for the [CXL Consortium](<https://computeexpresslink.org/>); ""they had the bad judgment to elect me to that"", he said with a grin. 

He had two different ways of looking at the development of famfs so far. It has been ""making things worse for VFS and FUSE since 2023"", but the filesystem community has been ""making perfect the enemy of good"" over that span as well. He hoped his session could help ""find a way to stop doing that"" so that famfs can make its way into the kernel. 

#### Introduction

As part of his introduction to famfs, Groves stressed that ""famfs cannot be used as a general-purpose filesystem"". An application cannot simply open a file on famfs and start writing to it; it is, instead, for creating non-sparse files in non-sparse, shareable memory. Famfs is not read-only, but is normally fairly static; data is moved into it and then accessed for analytics or other processing. 

Famfs provides a filesystem interface to shareable [DAX](<https://docs.kernel.org/filesystems/dax.html>) memory. Access is via regions created with [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>), while read and write are generally handled by [`memcpy()`](<https://man7.org/linux/man-pages/man3/memcpy.3.html>). 

[ ![\[John Groves\]](https://static.lwn.net/images/2026/lsfmb-groves-sm.png) ](<https://lwn.net/Articles/1082903/>)

Metadata is managed so that multiple hosts can mount a famfs filesystem; there is a single master node that puts the files into the filesystem to make them available to the client nodes. "File maps" describe the mapping of the file to offsets and ranges on multiple DAX devices. Interleaving data across devices is critical for performance; that's how memory is laid out for system RAM and it needs to be that way for shared memory devices as well. He said that the interleaving mechanism is not CXL-specific at all and could be used for other shared-memory devices that provide a DAX interface. 

Ted Ts'o asked whether the master node was the only node that could create or write files. Groves said that for the foreseeable future, only the master can create files, which are fully pre-allocated at creation time. It is usually only the master that writes to the files, as well, but clients can write ""and you're responsible for whether you're doing something that makes sense"". CXL ""makes cache coherency worse"", he said to chuckles around the room. 

There is no [device mapper](<https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/index.html>) for memory, Groves said; there are only page tables, translation lookaside buffers (TLBs), and fault handlers. A filesystem can bridge those worlds. But, famfs is not storage, just volatile memory—there is no backing store. Typically, the data of interest is collected into a file on a regular filesystem (XFS, say) by the master node, then copied to a famfs file. Each node in the cluster would then mount the famfs filesystem and run its workload on the data in the file. 

There are two separate implementations of famfs, the original standalone version and the more recent FUSE-based filesystem. He has taken the approach of supporting both implementations, which are differentiated by a mount-time option. The FUSE version came about after a discussion at the 2024 summit and it took him most of a year to get that working. 

There are 100TB appliances available now, though the memory for them may actually not be available until 2030, he said to knowing laughter. But there is currently no abstraction available to use them in the kernel. The ability to place enormous datasets into memory provides a means to solve problems that are currently impractical. Applications already know how to use files; with famfs, they do not need to be rewritten to take advantage of these new devices. 

Amir Goldstein reminded Groves that he should save some time ""for the contention point"", though Groves joked that he brought his beer so that he could not be deprived of it by a lengthy discussion as was postulated in the April LWN article. Eventually, Christian Brauner asked: ""what do you want from us?"" Essentially, the answer is "merge famfs", but it took Groves a bit of time to get there. 

The dilemma for the FUSE-based implementation is that famfs requires ABI changes to FUSE in order to handle its file maps. But the FUSE-based famfs does not perform as well as the standalone version. He has ""been slinging FUSE patches for a year"" at this point without much of a response from the FUSE community. Recently, though, there have been objections to the changes needed to the FUSE ABI. 

#### BPF?

As part of the discussion about those ABI changes, it was suggested that BPF be used to calculate the extents based on the interleave information. That would require running a BPF program as a fault handler, which is a clever idea, Groves said, but famfs is performance-critical, so it is not a good candidate to test whether the idea is viable. He feels like he keeps hearing that if he changes X, the FUSE community might accept it, but it is clear that he is growing weary of the frequently shifting ground. 

Goldstein summarized the two FUSE ABI changes that famfs needs, one to determine which DAX devices are present and another to provide the file map to the FUSE server so that it can handle the faults correctly. He asked Miklos Szeredi, who is the FUSE maintainer, if those were acceptable. Szeredi said that he was not opposed to merging famfs as it stands. As he [said](<https://lwn.net/ml/all/CAJfpegvVTcV89=q3L326aGQjhduBcv7PVg5QKftGLjNZmCLmaw@mail.gmail.com/>) in the [discussion of the FUSE famfs version 10 patch set](<https://lwn.net/ml/all/0100019d43e5f632-f5862a3e-361c-4b54-a9a6-96c242a8f17a-000000@email.amazonses.com/>), he would prefer to have an alternative to a famfs-specific interface. At the time, famfs was flying under his radar to a certain extent, so he had not looked in detail at what was being proposed for it; the BPF idea was raised and he is curious about how it would work. 

Christoph Hellwig objected to the whole idea of bringing BPF into the picture. He said that the problem with famfs really does not have much to do with the filesystem at all. It is, instead, applying a mapping layer in FUSE that is separate from the one that other filesystems are using: [iomap](<https://lwn.net/Articles/1079415/>). Darrick Wong has been working on an [implementation of iomap for a FUSE-based ext4](<https://lwn.net/ml/linux-fsdevel/177188734044.3935739.1368557343243072212.stgit@frogsfrogsfrogs/>) that takes the right approach. ""We want a user-space FUSE server to do a mapping of a logical offset into a physical layout either on a disk or on DAX and it totally makes sense to include striping."" He strongly suggested building a ""proper high-level infrastructure"" and not hiding this kind of work in filesystem-specific code. 

Groves said that he and Wong had spoken months earlier about combining their work, but they came to the conclusion that there are famfs-specific repeating patterns that are used for interleaving the data, which is not something that other filesystems need to do. 

An attendee asked about the performance difference for FUSE-based famfs versus standalone. Groves said that he has measured some things, but knows that FUSE is not noted for its performance. Goldstein pointed out that transferring the file map is not particularly performance-sensitive, which Groves acknowledged. 

Groves had attempted to run a proof of concept of the BPF idea, which Gregory Price had put together with an LLM's "assistance", but was unable to get it to work. Price agreed that the build environment used by the LLM was non-standard and that mostly what he took away from the experiment was that BPF did not make sense to use for FUSE. 

Goldstein said that if there is a way to describe the file maps that Szeredi is willing to add to the FUSE protocol, how it gets implemented right now does not really matter. It could use Groves's code now and iomap later, for example. This is why it is important to generalize new features, Hellwig said; Wong's current ext4 code does not do striping, but other kernel filesystems do (e.g. Btrfs), so that feature will be needed if the FUSE iomap code is to be used more widely. 

Hellwig suggested that a combined effort from Wong and Groves that added useful infrastructure for more than just the famfs use case might find a much easier path into the kernel. It sends a different message that shows other developers that the feature affects more than just famfs. Szeredi said that Wong and Groves had already tried to find common ground, but failed. 

At that point, Wong chimed in over the remote link. He said that the famfs pattern-based extents could result in huge numbers of mappings that would need to be stored by the kernel in an iomap-based implementation. A file of 100TB with, say, 2MB stripes, results in a huge list of extents that would need to be stored in kernel memory. To him, it did not make sense to do that when famfs can simply calculate that information based on the pattern it is using. ""It seems kind of silly to me to go upload billions of mappings into the kernel, just because you don't want to have a little bit of executable code."" That was what led to the ""crazy BPF thing"", which he said ""was horrible"". 

Hellwig said that the formulas for striping patterns are well-known and long-established; there are three parameters that are used to describe them and adding custom executable code for them (e.g. BPF) should not be needed. Groves agreed and said that those parameters are basically what he had used for the FUSE file-maps protocol message. Goldstein said that as long as others could use the file-maps facility for their own purposes, it should be merged into FUSE; the protocol details of the message fields and such will still need to be worked out, however. 

Wong suggested that he could take Groves's file-map patches and rename them to "FUSE map" or something generic to get them upstream with his FUSE iomap work. Goldstein agreed that something like that should be possible. Groves jokingly wondered if that meant he would need to wait until 2028 or so, but Brauner pointed out that someone had just offered to do much of the work for him—to more laughter. Goldstein said that it may seem like Wong's patches complicate what Groves is trying to do, but there is a fair amount of overlap and a lot of code that Groves can use for famfs. In the end, progress was made, Goldstein said and Groves agreed, so a path forward for famfs has hopefully been found at this point.
