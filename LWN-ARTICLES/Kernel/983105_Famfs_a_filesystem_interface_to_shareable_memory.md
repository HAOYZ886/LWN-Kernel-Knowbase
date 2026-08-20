---
title: "Famfs: a filesystem interface to shareable memory"
url: https://lwn.net/Articles/983105/
date: "July 30, 2024"
category: "Compute Express Link CXL; DAX; Filesystems-famfs"
author: "By Jake Edge July 30, 2024 LSFMM+BPF"
---

By **Jake Edge**  
July 30, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

At the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), John Groves led a session on famfs, which is a filesystem he has developed that uses the kernel's [direct-access (DAX)](<https://docs.kernel.org/filesystems/dax.html>) mechanism to access memory that is shareable between hosts. The discussion was aimed at whether a different approach should be taken and, in particular, whether FUSE should be used instead of implementing as an in-kernel filesystem. As noted in the [thread](<https://lwn.net/ml/all/20240229002020.85535-1-john@groves.net/>) about his proposal for an LSFMM+BPF session, and the mailing-list discussions on [the first](<https://lwn.net/ml/all/cover.1708709155.git.john%40groves.net/>) and [second version](<https://lwn.net/ml/linux-fsdevel/cover.1714409084.git.john@groves.net/>) of his patch set, there is some skepticism that a new in-kernel filesystem is warranted for the use case. 

#### Background

Groves said that he works for a memory vendor that is trying to enable the use of shareable memory; the "fam" stands for "fabric-attached memory". The way to think about that, he said, "is taking what `MAP_SHARED` does on a single host and making it possible to scale that out with shared memory". That would allow processes on multiple hosts to map the same shareable memory. The memory would presumably be [Compute Express Link](<https://en.wikipedia.org/wiki/Compute_Express_Link>) (CXL) memory, but nothing in famfs is CXL-specific, he said, famfs just uses DAX and DAX devices that are shared. 

[ ![\[John Groves\]](https://static.lwn.net/images/2024/lsfmb-groves-sm.png) ](<https://lwn.net/Articles/983713/>)

He gave a quick introduction to shareable memory; it is memory that more than one host can see, but it does not make sense to attach it as system RAM, because system RAM gets zeroed when it is brought online. So shareable memory will be available in the form of shared DAX devices. A shared DAX device is a low-level abstraction that few software applications know how to use. But lots of applications know how to use memory-mapped files with [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) and the `MAP_SHARED` flag, so the idea is to enable those applications to use shareable memory. Prior shareable-memory implementations have generally added new abstractions to support this kind of memory, but applications tend not to adopt them. 

Famfs is "a filesystem but it's really memory not storage"; it uses DAX devices but presumes they are not persistent. It is effectively an allocator where the allocations look like files, but it is not a general-purpose allocator; for example, it uses 2MB granularity and enforces huge-page alignment. 

The metadata of the files is handled by famfs in a way that allows the allocations to be shared between systems. He "cribbed a lot of code" from XFS, which is a filesystem that uses the fsdax mode, especially for the read, write, and `mmap()` paths. Famfs has a superblock and the file metadata is stored in an append-only log of the allocation and file-creation operations; the rest of the memory is referenced by files once they have been allocated and created. All of the files have a preallocated size, so there is no allocation done at write time. "We are not trying to be a filesystem, except that the filesystem is where the `mmap()` interface is." 

The first version of the patch set could either use a character `/dev/dax` device (devdax) or a block-based persistent-memory (pmem) device (fsdax), but he dropped the pmem interface for version 2\. Using a DAX device requires adding iomap support for it, so some of the preliminary patches in his series do so. Devdax handles the reads, writes, and page faults, requiring an upcall to user space for mapping the file offset to an offset on the DAX device. 

#### Why device DAX?

Jan Kara asked why the DAX device could not be exposed as a pmem device, which is what filesystems currently target. The difference is between using a character device, which is what CXL exposes, and a block device, which is what pmem devices are, Groves said. Kara said that he wondered because the filesystems that currently support DAX do so via the pmem interfaces; famfs needed to replicate a bunch of code to work with character devices. Groves said that there was not all that much code that was needed to make it work; in addition, pmem devices can be converted to DAX, but the reverse is not true. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Over the years, there has been some discussion about whether it makes sense to be able to run a filesystem using the character-device interface, Groves said. Dan Williams said that the same question came up when the original work on pmem was being done; it is not really a block device, but a byte-addressable memory range. "Block devices come with baggage that don't make sense for a memory device." The reason that devdax came about was to essentially throw away the pieces that did not make sense for a memory range. 

Kara said that he did not see a problem with filesystems going straight to devdax, rather than using pmem, but the existing DAX filesystems do not do that. Groves said that his patches would provide a path to change that, but Williams was surprised that might be on the table. Kara said that it would not be, initially, but that maintaining two sets of infrastructure did not make long-term sense. He would like to see some convergence so that all of the DAX filesystems used the same infrastructure, though he did not have a strong opinion on which (devdax- or pmem-based) should be used. There is already some precedent for filesystems that attach to character devices, he said, such as the [MTD](<https://en.wikipedia.org/wiki/Memory_Technology_Device>) filesystems. 

Groves said that the code in question is "quite small"; famfs is around 1000 lines of kernel code. It mostly consists of inode operations that are "very ramfs-like". The files are special, since they have the DAX flag set and their metadata is cached so that page faults can be handled, but that is mostly done in user space, which performs the space allocation, log-append, and log-play operations. The log-play operation reads the log and creates the files that it says should exist, but that information is ephemeral metadata in memory, which is never written to storage because there is nothing to write inodes back to. 

#### FUSE?

There are a number of companies testing famfs with various use cases. The main one is to "wrangle large data frames into a memory-efficient, mmap-able format" for programs like [Apache Arrow](<https://arrow.apache.org/>). Those files are put into famfs so that a "distributed futures orchestrator" like [Ray](<https://docs.ray.io/en/latest/index.html>) can allow multiple jobs to consume the data in a distributed way. The customers already know how to do that using files, so they do not even need to know that these files are special. For those use cases, there is generally no need for cache coherency between the systems because the data is written once when it is created and then used read-only after that. So a question that comes up is: since user space is doing most of the work, could famfs be implemented using FUSE? 

Groves said that he did a talk ([YouTube video](<https://www.youtube.com/watch?v=aA_DgO95gLo>)) about famfs at the [2023 Linux Plumbers Conference](<https://lpc.events/event/17/>). He has been working on it for around a year at this point. Beyond the kernel-space code, which is small as mentioned, there are also around 5000 lines of user-space code. But Amir Goldstein said that he did not like the lines-of-code metric; the problem is in maintaining duplicate code. That new code has likely missed some "subtle corner cases" that are not reflected in the counts. It should instead be looked at from the perspective of what functionality is missing in FUSE or [virtiofs](<https://virtio-fs.gitlab.io/>). 

Groves acknowledged that concern and said that there are two key requirements for famfs; the current code is already meeting them. It must run at full memory speed, which means that page faults must be handled from in-kernel cached metadata. It also must distribute metadata in a way that can be shared; users want to dump large datasets into famfs, such as data frames or indexes that can be searched after mapping the file. Currently, famfs does not even support delete, though that could be added in a limited way, but it does not reallocate any of the space in the filesystem. 

He is no FUSE expert, he said, but had some exchanges with Miklos Szeredi about virtiofs and its use of FUSE; as far as Groves can tell, virtiofs does not fit for famfs. He believes that FUSE could be used directly for famfs, but the 1000 lines of code would just move from famfs to FUSE, effectively. FUSE would need to allow famfs to cache an extent list; right now, a famfs instance uses a single DAX device but he foresees that changing eventually, so that a famfs could span more than one device. 

Goldstein said that virtiofs just maps memory from the guest to the host and simply provides an interface to do that. Over the remote audio interface, Szeredi said that he did not think it would be difficult to add what famfs needed to FUSE; it is a special kind of FUSE passthrough for the famfs files. Adding the support for extents would be useful for other use cases too. Williams asked if it would be able to work with multiple hosts accessing the shared memory and Szeredi said that it could. It probably does not matter if it is FUSE or famfs that is used to implement it, Groves said, but famfs is always going to need to process its own metadata. 

James Bottomley asked if most of the sharing is handled by DAX, with famfs just being a thin layer on top of that. Williams said that DAX is just a memory range, and that all of the properties of "how it is shared and whether it is shared would come from famfs". It provides a way to have shared metadata, Groves said, "and I punted on most of the hard problems". The metadata is all managed in user space, which is why people are talking about FUSE for famfs, Bottomley said. If there are just a few interfaces that need to be added to FUSE, that's probably the easier path. Groves is not opposed to using FUSE, though he is "slightly dubious" that the two are actually all that similar. 

Kara said that he did not know how feasible it was to use FUSE, that was best left to Goldstein and Szeredi, but he was concerned that famfs was heading down the well-trod path of starting with something simple that will only get more complicated due to feature creep. First it might be adding deletion, then users will ask for hole punching, and so on; this seems to always happen to the "simple" filesystems that are proposed. If it gets merged, it will not be so simple in five years. 

The second thing that worries Kara is what the testing picture looks like for a filesystem that is mostly implemented in user space. When filesystem interfaces change, that will affect famfs, so it will need to be tested, but its setup is all done in user space, which is non-standard for kernel filesystems. He would like to see "a good story of how other filesystem developers can work" with famfs. The advantage of FUSE is that the other filesystem maintainers do not have to worry unless they are changing the FUSE interfaces directly. This makes him lean toward seeing famfs as a FUSE filesystem. 

Groves said that it is pretty straightforward to test famfs, though it does require setting up a DAX device, which can be done with a kernel command-line option. The shared part is a bit more tricky, but can be tested using multiple virtual machines; he has real shared memory in his lab but still does a lot of testing with virtual machines. Goldstein noted that the session had run out of time, but suggested that Groves use the existing subsystems and if he encounters problems in doing so that he bring them up on the mailing lists. 

A [YouTube video](<https://www.youtube.com/watch?v=nMaZhXJJgmU>) of the session is available.
