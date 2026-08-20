---
title: The OrangeFS distributed filesystem
url: https://lwn.net/Articles/643165/
date: "May 6, 2015"
category: "Filesystems-OrangeFS"
author: "By Jake Edge May 6, 2015 Vault 2015"
---

By **Jake Edge**  
May 6, 2015

* * *

[Vault 2015](<https://lwn.net/Archives/ConferenceByYear/#2015-Vault>)

There is no shortage of parallel, distributed filesystems available in Linux today. Each have their strengths and weaknesses, as well as their advocates and use cases. [Orange File System](<http://www.orangefs.org/>) (or OrangeFS) is another; it is targeted at providing high I/O performance on systems with up to several thousand multicore storage nodes, but the project is planning to support millions of cores eventually. The OrangeFS client code was [proposed](<http://permalink.gmane.org/gmane.linux.file-systems/92078>) for the Linux kernel back in January. Walt Ligon, one of the principals behind the filesystem, gave a talk about OrangeFS at the [Vault conference](<http://events.linuxfoundation.org/events/vault>) back in March. 

At the beginning of the talk, Ligon noted that OrangeFS was similar "in some ways" to [GlusterFS](<http://www.gluster.org/>), which was the subject of an earlier [Vault presentation](<https://lwn.net/Articles/637437/>). But OrangeFS grew out of a research project from 1993 called [Parallel Virtual File System](<http://en.wikipedia.org/wiki/Parallel_Virtual_File_System>) (PVFS). That filesystem (now in version 2, called PVFS2) is in use today by various commercial organizations as well as by universities. In 2008, the PVFS project was renamed to OrangeFS as part of changing its focus to a more general filesystem for "big data". 

#### Overview

At its core, OrangeFS has a client-server architecture, most of which runs in user space. All of the code is available under the LGPL. There are multiple ways for client systems to use the PVFS protocol to access data on the servers. That includes libpvfs2 for low-level access, [MPI-IO](<http://en.wikipedia.org/wiki/Message_Passing_Interface#I.2FO>), [Filesystem in Userspace](<http://fuse.sourceforge.net/>) (FUSE), web-related mechanisms (e.g. [WebDAV](<http://en.wikipedia.org/wiki/WebDAV>), [REST](<http://en.wikipedia.org/wiki/Representational_state_transfer>)), and a Linux virtual filesystem (VFS) client implementation for mounting OrangeFS like any other filesystem in Linux. The latter is what is being proposed for upstream inclusion. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

OrangeFS servers handle objects, called dataspaces, that can have both byte-stream and key-value components. The "Trove" subsystem determines how to store those components. Currently, the byte streams are stored as files on the underlying filesystem, while the key-value data is mostly stored in [Berkeley DB](<http://en.wikipedia.org/wiki/Berkeley_DB>) files, though there is starting to be some use of [LMDB](<http://en.wikipedia.org/wiki/Lightning_Memory-Mapped_Database>). 

[ ![\[File structure diagram\]](https://static.lwn.net/images/2015/vault-orangefs-diag-sm.png) ](<https://lwn.net/Articles/643411/>) As seen in the diagram at right, files are stored as a collection of objects: a metadata object and one or more distributed file ("Dfile") objects. Those are accessed from directory objects that include a metadata object. Each of those point to various DirData objects, which contain Dirent (directory entry) objects that point to the metadata object of a file. 

Instead of blocks, OrangeFS is all about objects and leaves the block mapping to the underlying filesystems. There are no metadata servers, as all servers can handle all kinds of requests. It is possible to configure an OrangeFS filesystem to store its metadata separate from its data using parameters that govern how the objects should be distributed. Files are typically striped across multiple servers to facilitate parallel access. 

OrangeFS provides a unified namespace, so that all files are accessible from a single mount point. It has a client protocol that supports lots of parallel clients and servers. That provides "high aggregate throughput", Ligon said. 

In the past, users wanted MPI-IO access to files, but that has changed. Now, POSIX access is "what everyone wants to use". They want to be able to write Python scripts to access their data. But the POSIX API "can be a real limiting factor" because it doesn't understand parallel files, striping, and so on. 

Another of the goals for OrangeFS is to "enable the future" by being flexible about the underlying technologies it uses. It wants to provide ways to swap in new redundancy, availability, and stability techniques. For example, OrangeFS is designed to allow users to use their own distribution equation, which is used to find and store data. That equation allows the system to determine which servers go with each object. 

Another goal is to make OrangeFS grow to "[exascale](<http://en.wikipedia.org/wiki/Exascale_computing>)". One way to keep increasing storage is to add more disks to the computer, but that will eventually hit a wall. There is not enough bandwidth and compute [ ![\[Walt Ligon\]](https://static.lwn.net/images/2015/vault-ligon-sm.jpg) ](<https://lwn.net/Articles/643275/>) power within a system to access all that data with reasonable performance; the solution to that problem is to add more computers into the mix. 

That dramatically increases the number of cores accessing the data, but you can only increase the amount of storage per server to a certain point. Just as with the single computer system above, various limits will be hit, so a better solution is to add more servers with more network connections, but that can get costly. In an attempt to build a lower-cost alternative, Ligon has a new project to create, say, 500 storage servers, each using a Raspberry Pi with a disk. It will be much cheaper, but he thinks it will also be faster—though he still needs to prove that. 

There are a number of planned OrangeFS attributes that are missing from the discussion so far, he said. For example, with a large enough number of servers and disks, there will be failures every day. Even if there are no failures, systems will need to be taken down to update the operating system or other software, so there is a need for features that provide availability. 

Security is a "major issue" that has mostly been dealt with using "chewing gum and string", Ligon said. Data integrity is another important attribute, as the stored data must be periodically checked and repaired. There is also a need for ways to redistribute files and objects for load or space reasons, as well as a need for monitoring and administration tools. 

#### OrangeFS V3

Some of the "core values" for the next major version of OrangeFS (3.0 or V3) are directly targeted at solving those problems. At the top of that list is "parallelism"; the filesystem should allow parallel access to files, directories, and metadata, while providing scalability through adding servers. The filesystem should also recognize that things are going to fail regularly. If a copy goes bad, throw it away and recreate it; if a node fails, simply discard and replace it. 

OrangeFS V3 will minimize the dependencies between servers by not sharing state between them. That will allow servers to be added and removed as needed. Avoiding locks is key to providing better performance, which may require relaxing the semantics of some operations. Finally, 3.0 will target flexible site-customization policies for things like object placement, replication, migration, and so forth. 

In order to do all of that, OrangeFS will change the PVFS handle that has been used to identify objects. It is a 64-bit value that encodes both the object and the server it lives on. That scheme has a number of limitations. Objects cannot migrate or be replicated and the collection of servers is static. That works well up to around 128 static servers, he said, but it won't work for OrangeFS V3. 

The new handles will contain both an object ID and one or more server IDs, both of which will be 128-bit values. The number of server IDs will typically be somewhere between two and four that will be set when the filesystem is created; it can change, but in practice rarely will. These handles are internal-only, typically stored in metadata objects. By making this change, OrangeFS V3 will be able to do replication and migration. 

This will allow all of the filesystem structure to be replicated, as well as the file data. A set of back references is also created, so that maintenance operations can find other copies of the structures. Each of those pieces and copies could be stored on different servers if that was desired. Another possibility is to use "file stuffing", which places the first data object on the same server as its metadata object. 

Reads can be done from any server that has a copy of the object, while writes are done to the primary object. Its server then initiates the copy (or copies) needed for replication. The write will only complete and return to the client after a certain number of copies have completed. This is known as the "write confidence" required. For example, if one copy is sent to a much slower archive device, the write could complete after all or some of the non-archive copies have completed. 

V3 adds a server ID database, rather than a fixed set of servers. That allows dynamic addition of servers with site-defined attributes (e.g. number, building, rack, etc.). A client doesn't have to know about all the servers, only the set it is using. Servers maintain a partial list of other servers that they tend to work with and there is a server resolution protocol to find others as needed. 

The security model is already present in OrangeFS 2.9 (which is the current version of the filesystem). The model is based around capabilities that get returned based on the credentials presented when a metadata object is accessed. That capability is then passed when accessing the data objects. Certificates and public/private key pairs are used to authenticate clients and their credentials. 

The final OrangeFS feature that Ligon described was the "parallel background jobs" (PBJs) that are used for maintenance and data integrity. They can be run to check the integrity of the data stored and to repair problems that are found. They can also handle tasks like rebalancing where data is stored to avoid access hotspots and the like. 

As he said at the outset, Ligon's talk provided a high-level overview of the filesystem. It seems to not be a particularly well known filesystem, but one that has some interesting attributes. Beyond just handling large data sets for parallel computation, it is also targeted as a research platform that can be used to test ideas for enhancements or broad restructuring. The kernel patches did not receive any comments, but they are also fairly small (less than 10,000 lines of code), so it seems plausible that we will see an OrangeFS client land in the mainline sometime in the future. 

[I would like to thank the Linux Foundation for travel support to Boston for Vault.]
