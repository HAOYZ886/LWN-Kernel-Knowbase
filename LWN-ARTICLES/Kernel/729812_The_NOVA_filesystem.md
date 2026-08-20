---
title: The NOVA filesystem
url: https://lwn.net/Articles/729812/
date: "August 4, 2017"
category: "Filesystems-Nonvolatile memory"
author: "By Jonathan Corbet August 4, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 4, 2017

Nonvolatile memory offers the promise of fast, byte-addressable storage that persists over power cycles. Taking advantage of that promise requires the imposition of some sort of directory structure so that the persistent data can be found. There are a few approaches to the implementation of such structures, but the usual answer is to employ a filesystem, since managing access to persistent data is what filesystems were created to do. But traditional filesystems are not a perfect match to nonvolatile memory, so there is a natural interest in new filesystems that were designed for this media from the beginning. The recently posted [NOVA filesystem](<https://lwn.net/Articles/729770/>) is a new entry in this race. 

The filesystems that are currently in use were designed with a specific set of assumptions in mind. Storage is slow, so it is worth expending a considerable amount of CPU power and memory to minimize accesses to the underlying device. Rotational storage imposes a huge performance penalty on non-sequential operations, so there is great value in laying out data consecutively. Sector I/O is atomic; either an entire sector will be written, or it will be unchanged. All of these assumptions (and more) are wired deeply into most filesystems, but they are all incorrect for nonvolatile memory devices. As a result, while filesystems like XFS or ext4 can be sped up considerably on such devices, the chances are good that a filesystem designed from the beginning with nonvolatile memory in mind will perform better and be more resistant to data corruption. 

NOVA is intended to be such a filesystem. It is not just unsuited for regular block devices, it cannot use them at all, since it does not use the kernel's block layer. Instead, it works directly with storage mapped into the kernel's address space. A filesystem implementation gives up a lot if it avoids the block layer: request coalescing, queue management, prioritization of requests, and more. On the other hand, it saves the overhead imposed by the block layer and, when it comes to nonvolatile memory performance, cutting down on CPU overhead is a key part of performing well. 

#### NOVA filesystem structure

Like most filesystems, NOVA starts with a superblock — the top-level data structure that describes the filesystem and provides the locations of the other data structures. One of those is the inode table, an inode being the internal representation of a file (or directory) within the filesystem. The NOVA inode table is set up as a set of per-CPU arrays, allowing any CPU to allocate new inodes without having to take cross-processor locks. 

Free space is also split across a system's CPUs; it is managed in a [red-black tree](<https://lwn.net/Articles/184495/>) on each processor to facilitate coalescing of free regions. Unlike the inode tables, the free lists are maintained in normal RAM, not nonvolatile memory. They are written back when the filesystem is unmounted; if the filesystem is not unmounted properly, the free list will be rebuilt with a scan of the filesystem as a whole. 

Perhaps the most interesting aspect of NOVA is how the inodes are stored. On a filesystem like ext4, the on-disk inode is [a well-defined structure](<http://elixir.free-electrons.com/linux/v4.12/source/fs/ext4/ext4.h#L716>) containing much of a file's metadata. To make things fast, NOVA took a different approach based on log-structured filesystems. The result is that an inode in the table is just a pair of pointers to a data structure that looks something like this: 

> ![\[inode log\]](https://static.lwn.net/images/2017/nova-inode.png)

Each active inode consists of a log describing the changes that have been made to the file; all that is found in the inode structure itself is a pair of pointers indicating the first and last valid log entries. Those entries are stored (in nonvolatile memory) in chunks of at least 4KB, organized as a linked list. Each log entry will indicate an event like: 

  * The attributes of the file have been changed — a change in the permission bits, for example. 

  * An entry has been added to a directory (for directory inodes, obviously). 

  * A link to the file was added. 

  * Data has been written to the file. 

The case of writing data is worth looking at a bit more closely. If a process writes to an empty file, there will be no data pages already allocated. The NOVA implementation will allocate the needed memory from the per-CPU free list and copy the data into that space. It will then append an entry to the inode log indicating the new length of the file and pointing to the location in the array where the data was written. Finally, an atomic update of the inode's tail pointer will complete the operation and make it visible globally. 

If, instead, a write operation overwrites existing data, things are done a little differently. NOVA is a copy-on-write (COW) filesystem, so the first step is, once again, to allocate new (nonvolatile) memory for the new data. Data is copied from the old pages into the new if necessary, then the new data is added. A new entry is added to the log pointing to the new pages, the tail pointer is updated, and the old log entry for those pages is invalidated. At that point, the operation is complete and the old data pages can be freed for reuse. 

Thus, the "on-disk" inode in NOVA isn't really a straightforward description of the file it represents. It is perhaps better thought of as a set of instructions that, when followed in order (and skipping the invalidated ones) will yield a complete description of the file and its layout in memory. This structure has the advantage of being quite fast to update when the file changes, with minimal locking required. It will obviously be a bit slower when it comes to accessing an existing file. NOVA addresses that by assembling a compact description of the file in RAM when the file is opened. Even that act of assembly should not be all that slow. Remember that the whole linked-list structure is directly addressable by the CPU. Storing this type of structure on a rotating disk, or even on a solid-state disk accessed as a normal block device, would be prohibitively slow, but direct addressability changes things. 

There is another interesting feature enabled by this log structure. Each entry in the log contains an "epoch number" that is set when the entry is created. That makes it possible to create snapshots by incrementing the global epoch number, and associating the previous number with a pointer to the snapshot. When the snapshot is mounted, any log entries with an epoch number greater than the snapshot's number can be simply ignored to give a view of the file as it existed when the snapshot was taken. There are some details to manage, of course: entries associated with snapshots cannot be invalidated, and those entries have to be passed over when the snapshot is not in use. But it is still an elegant solution to the problem. 

#### DAX and beyond

Readers may be wondering about how NOVA interacts with the kernel's DAX interface, which exists to allow applications to directly map files in nonvolatile memory into their address space, bypassing the kernel entirely for future accesses. It can be hard to make direct mapping work well with a COW-based write mechanism. In [this 2016 paper describing NOVA [PDF]](<http://cseweb.ucsd.edu/~swanson/papers/FAST2016NOVA.pdf>), the authors say they don't even try. Rather than support DAX, NOVA supports an alternative mechanism called "atomic mmap" which copies data into "replica pages" and maps those instead. In a sense, atomic mmap reimplements a part of the page cache. 

One can imagine that this approach was seen as being suboptimal; direct access to nonvolatile memory is one of that technology's most compelling features. Happily, the posted patch set does claim to support DAX. As far as your editor can tell from the documentation and the code, NOVA disables COW for the portions of a file that have been mapped into a process's address space, so changes are made in place. One significant shortcoming is that pages that have been mapped into a process's address space cannot be written to with `write()`. There is some relatively complex logic (described in [this other paper [PDF]](<http://cseweb.ucsd.edu/~swanson/papers/TechReport2017HardenedNOVA.pdf>)) to ensure that the filesystem does the right thing when taking a snapshot of a file that is currently directly mapped into some process's address space. 

There are a number of self-protection measures built into NOVA, including checksumming for data and metadata. One of the more interesting mechanisms seems likely to prove controversial, though. One possible hazard of having your entire storage array mapped into the kernel's address space is that writing to a stray pointer can directly corrupt persistent data. That would not be a concern in a bug-free kernel but, well, that is not the world we live in. In an attempt to prevent inadvertent overwriting of data, NOVA can keep the entire array mapped read-only. When a change must be made, the processor's write-protect bit is temporarily cleared, allowing the kernel to bypass the memory permissions. Disabling write protection has been [deemed too dangerous](<https://lwn.net/Articles/724319/>) in the past; it seems unlikely that the idea will get a better reception now. Protection against stray writes is a valuable feature, though, so hopefully another way to implement it can be found. 

There are a few other things that will need to be fixed before NOVA can be seriously considered for merging upstream. For example, it only works on the x86-64 architecture and, due to the per-CPU inode table structure, it is impossible to move a NOVA filesystem from one system to another if the two machines do not have the same number of CPUs. NOVA doesn't support access control lists or disk quotas. There is no filesystem checker tool. And so on. The developers are aware of these issues and expect to deal with them. 

The fact that the developers do want to take care of those details and get the filesystem upstream is generally encouraging, but it is especially so given that NOVA comes from the academic world (from the University of California at San Diego in particular). Academic work has a discouraging tendency to stop when the papers are published and the grant money runs out, so the free-software world in general gets far less code from universities than one might expect. With luck, NOVA will be one development that escapes academia and becomes useful to the wider world. 

There are, of course, many other aspects of this filesystem that cannot be covered in such a short article. See the two papers referenced above and [the documentation in the patch itself](<https://lwn.net/Articles/729920/>) for more information. This appears to be a project to keep an eye on; if all goes well, it will show the way forward for getting full performance out of the huge, fast nonvolatile memory arrays that, we're told, we'll all be able to get sometime soon.
