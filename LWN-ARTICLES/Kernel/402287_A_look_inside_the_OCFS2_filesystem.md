---
title: A look inside the OCFS2 filesystem
url: https://lwn.net/Articles/402287/
date: "September 1, 2010"
category: "Clusters-Filesystems; Filesystems-Cluster"
author: "September 1, 2010 This article was contributed by Goldwyn Rodrigues"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

September 1, 2010

This article was contributed by Goldwyn Rodrigues

The [Oracle Cluster Filesystem (ocfs2)](<http://oss.oracle.com/projects/ocfs2/>) is a filesystem for clustered systems accessing a shared device, such as a Storage Area Network (SAN). It enables all nodes on the cluster to see the same files; changes to any files are visible immediately on other nodes. It is the filesystem's responsibility to ensure that nodes do not corrupt data by writing into each other's files. To guarantee integrity, ocfs2 uses the Linux [Distributed Lock Manager (DLM)](<http://en.wikipedia.org/wiki/Distributed_lock_manager>) to serialize events. However, a major goal of a clustered filesystem is to reduce cluster locking overhead in order to improve overall performance. This article will provide an overview of ocfs2 and how it is structured internally. 

###  A brief history 

Version 1 of the ocfs filesystem was an early effort by Oracle to create a filesystem for the clustered environment. It was a basic filesystem targeted to support Oracle database storage and did not have most of the POSIX features due to its limited disk format. Ocfs2 was a development effort to convert this basic filesystem into a general-purpose filesystem. The ocfs2 source code was merged in the Linux kernel with 2.6.16; since this merger (in 2005), a lot of features have been added to the filesystem which improve data storage efficiency and access times. 

###  Clustering 

Ocfs2 needs a cluster management system to handle cluster operations such as node membership and fencing. All nodes must have the same configuration, so each node knows about the other nodes in the cluster. There are currently two ways to handle cluster management for ocfs2: 

  * **Ocfs2 Cluster Base (O2CB)** \- This is the in-kernel implementation of cluster configuration management; it provides only the basic services needed to have a clustered filesystem running. Each node writes to a heartbeat file to let others know the node is alive. More information on running the ocfs2 filesystem using o2cb can be found int [the ocfs2 user guide [PDF]](<http://oss.oracle.com/projects/ocfs2/dist/documentation/v1.4/ocfs2-1_4-usersguide.pdf>). This mode does not have the capability of removing nodes from a live cluster and cannot be used for cluster-wide POSIX locks. 

  * **Linux High Availability** \- uses user-space tools, such as heartbeat and pacemaker, to perform cluster management. These packages are complete cluster management suites which can be used for advanced cluster activities such as different fail-over, STONITH (Shoot The Other Node In The Head - yes, computing terms can be barbaric), migration dependent services etc. It is also capable of removing nodes from a live cluster. It supports cluster-wide POSIX locks, as opposed to node-local locks. More information about cluster management tools can be found at [clusterlabs.org](<http://clusterlabs.org/>) and [linux-ha.org](<http://linux-ha.org/>) 

###  Disk format 

ocfs2 separates the way data and metadata are stored on disk. To facilitate this, it has two types of blocks: 

  * Metadata or simply "blocks" - the smallest addressable unit. These blocks contain the metadata of the filesystem, such as the inodes, extent blocks, group descriptors etc. The valid sizes are 512 bytes to 4KB (incremented in powers of two). Each metadata block contains a signature that says what the block contains. This signature is cross-checked while reading that specific data type. 

  * Data Clusters - data storage for regular files. Valid cluster sizes range from 4KB to 1MB (in powers of two). A larger data cluster reduces the size of filesystem metadata such as allocation bitmaps, making filesystem activities such as data allocation or filesystem checks faster. On the other hand, a large cluster size increases [internal fragmentation](<http://en.wikipedia.org/wiki/Internal_fragmentation#Internal_fragmentation>). A large cluster size is recommended for filesystems storing large files such as virtual machine images, while a small data cluster size is recommended for a filesystem which holds lots of small files, such as a mail directory. 

####  Inodes 

An inode occupies an entire block. The block number (with respect to the filesystem block size) doubles as the inode number. This organization may result in high disk space usage for a filesystem with a lot of small files. To minimize that problem, ocfs2 packs the data files into the inode itself if they are small enough to fit. This feature is known as "inline data." Inode numbers are 64 bits, which gives enough room for inode numbers to be addressed on large storage devices. 

![Ocfs2 inode layout](https://static.lwn.net/images/ns/kernel/ocfs2/inode.png) Data in a regular file is maintained in a B-tree of extents; the root of this B-tree is the inode. The inode holds a list of extent records which may either point to data extents, or point to extent blocks (which are the intermediate nodes in the tree). A special field called `l_tree_depth` contains the depth of the tree. A value of zero indicates that the blocks pointed to by extent records are data extents. The extent records contain the offset from the start of the file in terms of cluster blocks, which helps in determining the path to take while traversing the B-tree to find the block to be accessed. 

The basic unit of locking is the inode. Locks are granted on a special DLM data structure known as a [lock resource](<http://en.wikipedia.org/wiki/Distributed_lock_manager#Resources>). For any access to the file, the process must request a DLM lock on the lock resource over the cluster. DLM offers six [lock modes](<http://en.wikipedia.org/wiki/Distributed_lock_manager#Lock_Modes>) to differentiate between the type of operation. Out of these, ocfs2 uses only three: exclusive, protected read, and null locks. The inode maintains three types of lock resources for different operations: 

  * **read-write lock resource** : is used to serialize writes if multiple nodes perform I/O at the same time on a file. 

  * **inode lock resource** : is used for metadata inode operations 

  * **open lock resource** : is used to identify deletes of a file. When a file is open, this lock resource is opened in protected-read mode. If the node intends to delete it, it will request for a exclusive lock. If successful, it means that no other node is using the file and it can be safely deleted. If unsuccessful, the inode is treated as an orphan file (discussed later) 

####  Directory 

Directory entries are stored in name-inode pairs in blocks known as directory blocks. Access to the storage pattern of directory blocks is the same as for a regular file. However, directory blocks are allocated as cluster blocks. Since a directory block is considered to be a metadata block, the first allocation uses only a part of the cluster block. As the directory expands, the remaining unused blocks of the data cluster are filled until the data cluster block is fully used. 

![ocfs2 directory
layout on disk](https://static.lwn.net/images/ns/kernel/ocfs2/directory.png) A relatively new feature is indexing the directory entries for faster retrievals and improved lookup times. Ocfs2 maintains a separate indexed tree based on the hash of the directory names; the hash index points to the directory block where the directory entry can be found. Once the directory block is read, the directory entry is searched linearly. 

A special directory trailer is placed at the end of a directory block which contains additional data about that block. It keeps a track of the free space in the directory block for faster free space lookup during directory entry insert operations. The trailer also contains the checksum for the directory block, which is used by the metaecc feature (discussed later). 

####  Filesystem Metadata 

A special system directory (`//`) contains all the metadata files for the filesystem; this directory is not accessible from a regular mount. Note that the `//` notation is used only for the debugfs.ocfs2 tool. Files in the system directory, known as system files, are different from regular files, both in the terms on how they store information and what they store. 

An example of a system file is the slotmap, which defines the mapping of a node in the cluster. A node joins a cluster by providing its unique DLM name. The slot map provides it with a slot number, and the node inherits all system files associated with the slot number. The slot number assignment is not persistent across boots, so a node may inherit the system files of another node. All node-associated files are suffixed by the slot number to differentiate the files of different slots. 

A global bitmap file in the system directory keeps a record of the allocated blocks on the device. Each node also maintains a "local allocations" system file, which manages chunks of blocks obtained from the global bitmap. Maintaining local allocations decreases contention over global allocations. 

The allocation units are divided into the following: 

  * **inode_alloc** : allocates inodes for the local node. 

  * **extent_alloc** : allocates extent blocks for the local node. Extent blocks are intermediate leaf nodes in the B-tree storage of the files. 

  * **local_alloc** : allocates data in data cluster sizes for the use of regular file data. 

![Allocator layout](https://static.lwn.net/images/ns/kernel/ocfs2/allocator.png) Each allocator is associated with an inode; it maintains allocations in units known as "block groups." The allocation groups are preceded by a group descriptor which contains details about the block group, such as free units, allocation bitmaps etc. The allocator inode contains a chain of group descriptor block pointers. If this chain is exhausted, group descriptors are added to the existing ones in the form of linked list. Think of it as an array of linked lists. The new group descriptor is added to the smallest chain so that number of hops required to reach an allocation unit is small. 

Things get complicated when allocated data blocks are freed because those blocks could belong to the allocation map of another node. To resolve this problem, a "truncate log" maintains the blocks which have been freed locally, but not yet returned to the global bitmap. Once the node gets a lock on the global bitmap, the blocks in the local truncate log are freed. 

A file is not physically deleted until all processes accessing the file close it. Filesystems such as ext3 maintain an orphan list which contains a list of files which have been unlinked but still are in use by the system. Ocfs2 also maintains such a list to handle orphan inodes. Things are a bit more complex, however, because a node must check that a file to be deleted is not being used anywhere in the cluster. This check is coordinated using the inode lock resource. Whenever a file is unlinked, and the removed link happens to be the last link to the file, a check is made to determine whether another node is using the file by requesting an exclusive lock over inode lock resource. If the file is being used, it will be moved to the orphan directory and marked with a OCFS2_ORPHANED_FL flag. The orphan directory is later scanned to check for files not being accessed by any of the nodes in order to physically remove them from the storage device. 

Ocfs2 maintains a journal to deal with unexpected crashes. It uses the Linux JBD2 layer for journaling. The journal files are maintained, per node, for all I/O performed locally. If a node dies, it is the responsibility of the other nodes in the cluster to replay the dead node's journal before proceeding with any operations. 

###  Additional Features 

Ocfs2 has a couple of other distinctive features that it can boast about. They include: 

  * **Reflinks** is a feature to support snapshotting of files using copy-on-write (COW). Currently, a system call interface, to be called [reflink()](<http://lwn.net/Articles/333783/>) or `copyfile()` is being discussed upstream. Until the system call is finalized, users can access this feature via the `reflink` system tool which uses an `ioctl()` call to perform the snapshotting. 

  * **Metaecc** is an error correcting feature for the metadata using Cyclic Redundancy Check (CRC32). The code warns if the calculated error-correcting code is different from the one stored, and re-mounts the filesystem read-only in order to avoid further corruption. It is also capable of correcting single-bit errors on the fly. A special data structure, ocfs2_block_check, is embedded in most metadata structures to hold the CRC32 values associated with the structure. 

Ocfs2 developers continue to add features to keep it up to par with other new filesystems. Some features to expect in the near future are delayed allocation, online filesystem checks, and defragmentation. Since one of the main goals of ocfs2 is to support a database, file I/O performance is considered a priority, making it one of the best filesystems for the clustered environment. 

[Thanks to Mark Fasheh for reviewing this article.]
