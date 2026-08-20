---
title: The Ceph filesystem
url: https://lwn.net/Articles/258516/
date: "November 14, 2007"
category: "Filesystems-Ceph"
author: "By Jake Edge November 14, 2007"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
November 14, 2007

Ceph is a distributed filesystem that is described as scaling from gigabytes to petabytes of data with excellent performance and reliability. The project is LGPL-licensed, with plans to move from a FUSE-based client into the kernel. This led Sage Weil to post a [message to linux-kernel](<http://lwn.net/Articles/258307/>) describing the project and looking for filesystem developers who might be willing to help. There are quite a few interesting features in Ceph which might make it a nice addition to Linux. 

Weil outlines why he thinks Ceph might be of interest to kernel hackers: 

I periodically see frustration on this list with the lack of a scalable GPL distributed file system with sufficiently robust replication and failure recovery to run on commodity hardware, and would like to think that--with a little love--Ceph could fill that gap.

The filesystem is well described in a [paper](<http://www.usenix.org/events/osdi06/tech/full_papers/weil/weil_html/index.html>) from the 2006 USENIX Operating Systems Design and Implementation conference. The project's [homepage](<http://ceph.sourceforge.net/>) has the expected mailing list, wiki, and source code repository along with a detailed overview of the feature set. 

Ceph is designed to be extremely scalable, from both the storage and retrieval perspectives. One of its main innovations is splitting up operations on metadata from those on file data. With Ceph, there are two kinds of storage nodes, metadata servers (MDSs) and object storage devices (OSDs), with clients contacting the type appropriate for the kind of operation they are performing. The MDSs cache the metadata for files and directories, journaling any changes, and periodically writing the metadata as a data object to the OSDs. 

Data objects are distributed throughout the available OSDs using a hash-like function that allows all entities (clients, MDSs, and OSDs) to independently calculate the locations of an object. Coupled with an infrequently changing OSD cluster map, all the participants can figure out where the data is stored or where to store it. 

Both the OSDs and MDSs rebalance themselves to accommodate changing conditions and usage patterns. The MDS cluster distributes the cached metadata throughout, possibly replicating metadata of frequently used subtrees of the filesystem in multiple nodes of the cluster. This is done to keep the workload evenly balanced throughout the MDS cluster. For similar reasons, the OSDs automatically migrate data objects onto storage devices that have been newly added to the OSD cluster; thus distributing the workload by not allowing new devices to sit idle. 

Ceph does N-way replication of its data, spread throughout the cluster. When an OSD fails, the data is automatically re-replicated throughout the remaining OSDs. Recovery of the replicas can be parallelized because both the source and destination are spread over multiple disks. Unlike some other cluster filesystems, Ceph starts from the assumption that disk failure will be a regular occurrence. It does not require OSDs to have RAID or other reliable disk systems, which allows the use of commodity hardware for the OSD nodes. 

In his linux-kernel posting, Weil describes the current status of Ceph: 

I would describe the code base (weighing in at around 40,000 semicolon-lines) as early alpha quality: there is a healthy amount of debugging work to be done, but the basic features of the system are complete and can be tested and benchmarked.

In addition to creating an in-kernel filesystem for the clients (OSDs and MDSs run as userspace processes), there are several other features - notably snapshots and security - listed as needing work. 

Originally the topic of Weil's PhD. thesis, Ceph is also something that he hopes to eventually use at a web hosting company he helped start before graduate school: 

We spend a lot of money on storage, and the proprietary products out there are both expensive and largely unsatisfying. I think that any organization with a significant investment in storage in the data center should be interested [in Ceph]. There are few viable open source options once you scale beyond a few terabytes, unless you want to spend all your time moving data around between servers as volume sizes grow/contract over time. 

Unlike other projects, especially those springing from academic backgrounds, Ceph has some financial backing that could help it get to a polished state more quickly. Weil is looking to hire kernel and filesystem hackers to get Ceph to a point where it can be used reliably in production systems. Currently, he is sponsoring the work through his web hosting company, though an independent foundation or other organization to foster Ceph is a possibility down the road. 

Other filesystems with similar feature sets are available for Linux, but Ceph takes a fundamentally different approach to most of them. For those interested in filesystem hacking or just looking for a reliable solution scalable to multiple petabytes, Ceph is worth a look.
