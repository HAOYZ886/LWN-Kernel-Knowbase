---
title: Cluster support for MD/RAID 1
url: https://lwn.net/Articles/674085/
date: "February 3, 2016"
category: "Block layer-RAID; RAID"
author: "February 3, 2016 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

February 3, 2016

This article was contributed by Neil Brown

A high-availability (HA) cluster needs high-availability storage. There are two broad approaches to achieving this requirement. One is for some (or all) of the nodes to have local storage and for each write request to be sent on the network to at least two of those nodes. This is the approach taken by [DRBD](<https://en.wikipedia.org/wiki/Distributed_Replicated_Block_Device>) and [Ceph](<https://en.wikipedia.org/wiki/Ceph_%28software%29>), among [others](<https://en.wikipedia.org/wiki/List_of_file_systems#Distributed_parallel_fault-tolerant_file_systems>). It is not the approach we will be exploring today.

The second approach is to have suitably redundant storage quite separate from the cluster nodes. It is not unusual for an HA cluster to have a storage-area-network (SAN), using Fibre Channel or iSCSI, that connects all nodes equally to a storage server. This server might have many devices configured internally as RAID 6 or RAID 10 or similar and may even have redundant power and networking. Despite this, it can still be a [single point of failure](<https://en.wikipedia.org/wiki/Clustered_file_system#Avoiding_single_point_of_failure>) and would not fit, for example, a disaster-management plan that required each of two separate sites to be able to function without the other.

An obvious approach to managing HA storage in this sort of scenario is to use host-based mirroring across two storage servers, possibly at different locations. As the mirroring is host-based, there is no RAID 1 controller that could be a single point of failure. However, there is some complexity in allowing the different nodes to work with the various storage servers without inconveniencing each other. Clustered MD/RAID 1 is being developed by engineers at SUSE, in particular by Goldwyn Rodrigues and Guoqing Jiang, as a solution for managing this complexity and providing seamless storage in a cluster using multiple mirrored devices attached to all nodes.

The other main entrant in this space is “[CLVM](<https://en.wikipedia.org/wiki/Logical_Volume_Manager_%28Linux%29#High_availability>)”, the clustered version of LVM2. CLVM can be used to ensure that when a number of devices are attached to multiple nodes in the cluster, any logical volume created or activated on one node is available on all other nodes. When a mirrored volume is activated, that volume can be accessed on all nodes, which provides the functionality being discussed here. A problem, though, is that performance is not as high as [some](<https://www.mail-archive.com/users@clusterlabs.org/msg01275.html>) would like. For reasons that will be explained once we get into the technical details, the synchronization overhead imposed by CLVM on RAID 1 can be prohibitive. Clustered MD/RAID 1 takes a different approach that early testing suggests has significantly reduced overhead.

#### The big picture: active-active with a cluster filesystem

The goal here is to have a cluster where all nodes can write to the same filesystem at the same time. Naturally, this requires a filesystem that understands the difficulties of working in a cluster and there are several such filesystems in existence -- such as [OCFS2](<https://en.wikipedia.org/wiki/OCFS2>) or [GFS2](<https://en.wikipedia.org/wiki/GFS2>). These filesystems are designed to work with shared storage where each node can potentially write directly to any block. They use a cluster-wide locking service such as [DLM](<https://en.wikipedia.org/wiki/Distributed_lock_manager>) (the Distributed Lock Manager) to coordinate between nodes. The locks and the allocation of storage space are generally designed to avoid unnecessary contention, so if two nodes are creating and writing files in different directories, that should largely be able to continue without any locking traffic. Each node claims, for example, a large amount of unused space in a single cluster request, and then allocates it locally to whatever file it chooses.

With a cluster filesystem in control, the underlying RAID 1 needs no extra coordination across the cluster for I/O. Writes simply go to both devices and reads are directed to either. The one possible issue that needs to be considered is the possibility of two nodes writing to the same block at the same time. If this happened, then the two devices could contain different data, so future reads might sometimes get one value and sometimes the other. This is a worse result than with a single storage device, where the result of future reads would still be undefined, but would at least be stable. MD could use locking to protect against this possibility, but it would introduce substantial overhead of exactly the sort that the cluster filesystem has been designed to avoid. Neither CLVM or clustered MD perform this locking. Instead, they simply assume that such concurrent writes never happen. 

With this assumption in place, MD does not need any cluster-wide communication as long as all the hardware is working correctly. Of course, it is when a failure happens that RAID becomes interesting and this is definitely the case in a cluster. The significant issue is what happens when a node fails or is unexpectedly disconnected from one or both storage devices. If the node had been writing to storage at the time something failed, the write could have succeeded on one device and not on the other, resulting in inconsistent content. This is not good. 

When a node fails, the cluster-management mechanisms will notify the filesystem and it will have its own protocols, such as log replay, for dealing with inconsistencies left behind. But it will assume that each block on the storage has just one value, so that reads from different nodes won't report different contents. To protect this assumption, the RAID 1 layer must ensure that any inconsistency between the two halves of the mirror is never visible.

MD/RAID 1 must already deal with this possibility on a single-node system; it does so by “resyncing” the devices after a crash and by only reading from a single device until the resync finishes. This resync can take a long time, so a bitmap of possibly-out-of-sync regions is stored. Bits are set before a write commences, and are cleared when there have been no writes for a while. Just resyncing the part of the array listed in this bitmap is much faster than resyncing the whole array, so the small cost of updating the bitmap is justified by the significantly faster recovery.

The dm-raid1 module used with CLVM has a similar technique, though the terminology is different. Dm-raid1 maintains a “dirty region log” that serves the same function as the bitmap in MD/RAID 1\. While they serve the same function, these two are managed quite differently in a clustered configuration, and this is probably the biggest difference between the CLVM/RAID 1 and clustered MD/RAID 1.

When CLVM is used, the “dirty region log” is managed by the “dm-log-userspace” module which handles all internal requests to mark or clear a region, or to test if a region is dirty, by sending a message to a user-space daemon. This daemon will replicate the information around the cluster using an appropriate protocol and may or may not store it on disk.

Clustered MD/RAID 1 continues to use a bitmap stored on both (or “all”) the devices in the array, but has a separate bitmap for each cluster node. Thus setting or clearing a bit (the equivalent of marking or clearing a region) proceeds identically to the single-node case — a simple write to all disks is sufficient.

Thus, to mark a region as “dirty”, CLVM must send a message to user space that must then be passed to all nodes in the cluster and acknowledgments must return to the originating node, and then to the kernel module. The same task in clustered MD/RAID 1 involves writing to a block of each storage device and waiting for confirmation. This difference is the main reason for expecting a speed improvement.

#### Walk-through of a node failure

Probably the best way to understand how failure is managed is to walk through the various steps involved. This walk-through begins with the cluster in a stable but busy state where all nodes have been writing to the filesystem that is replicated on two attached storage devices. Both of these devices have a set of bitmaps, one per node, and several bits in each bitmap are set to reflect the fact that there have been recent writes to corresponding regions and some of those writes may not yet have completed.

![\[Cluster\]](https://static.lwn.net/images/2016/cluster.png)  
**Four-node cluster  
with two storage devices.**

The walk-through starts with the failure of one node. The details of what goes wrong are not important, but somehow the cluster-management software determines that the node is no longer part of the cluster. Possibly, this will involve proactively disabling the node ([STONITH](<https://en.wikipedia.org/wiki/STONITH>)) so that it cannot possibly write anything else at all to either device.

The first that MD knows of this is that it is told that node recovery is happening (the DLM `recover_prep()` callback is called). At this point, MD doesn't know any details but it knows something is amiss. It disables read-balancing so it only reads from the “first” device in the array (all nodes have the same understanding of “first”). This ensures that once the filesystem is told the extra details and possibly starts reading blocks that the failed node was writing, it will never risk seeing inconsistent data.

Second, MD is told which node has failed. It responds by taking note that the bitmap might be involved in a resync (so read-balancing should still be avoided) and by trying to claim the content of the bitmap owned by that node. The first node to get a lock on the appropriate DLM resource will copy that bitmap into its own bitmap, clear the original, and advise other nodes that it has taken over. Once this advice is sent and received, the blanket disabling of read-balancing can be relaxed. 

At this point, service is largely back to normal. The filesystem will possibly still be doing some recovery but it can be sure it will always see consistent data. The out-of-sync regions still need to be resolved, though, so the node that copied the bitmap information will start a resync process after first claiming an exclusive lock on the “resync” resource, to ensure only one node is resyncing at a time.

#### Cluster resync

Resync involves reading from one device in the array (the “first”) and writing to all the others. If this is done while another node is performing a filesystem write to the same region, new corruption could occur. To avoid this, the resyncing node chooses a range of blocks and advises all other nodes that it is performing a resync. Other nodes will block new writes from commencing in this range and wait for pending writes to complete. When they acknowledge that they have stopped writing to the range, the region is resynced and the process moves forward. This is similar to the way resync happens with a single node; writes are suspended briefly while the resync makes progress. In a cluster, the higher communication overheads will likely result in larger regions being suspended and slightly longer delays for any writes that happen in the affected range.

As the resync progresses the other nodes are told about it a range at a time; those nodes can start read-balancing on the area that is known to be in-sync.

So far there have been two mentions of one node sending a message to the others: once when a node reports that it has taken over a particular bitmap, and once when that node advises others that it is resyncing a given range. DLM does not provide a general message-passing mechanism, but it provides two useful tools. One is a small (e.g. 64 byte) "Lock Value Block" (LVB) that can be attached to a resource for all nodes to see. The other is a callback that can be triggered on a node that has locked a resource and that is blocking some other node from locking the same resource. Clustered MD uses these on a set of three resources to provide a general broadcast facility and uses it in several different situations. 

#### Failed node re-integration

After the resync process has finished, the remaining nodes are back to normal, can read-balance across all devices, and are resilient against single device failure again. There is just one more step in the walk-through: the step where the failed node comes back online.

When a node comes online in a cluster, DLM assigns it an index number. This number is used as an index into the list of bitmaps in the RAID 1, so it immediately know where it needs to store its intent to write. Before it does that, it needs to make sure that it is up-to-date with any resync that is currently happening in the cluster. This ensures it doesn't read-balance where it shouldn't or write over an ongoing resync.

The LVBs play a role here too, being used to store the current state of any ongoing resync. The new node can read these to ensure it knows the current state, and then can listen for broadcast of RESYNCING messages to follow the progress through the resync. 

Once all these LVBs have been inspected, the new node will be able to follow any ongoing resync, knows which regions are safe to read-balance over, and can continue as a fully-fledged member of the cluster. This node's own bitmap will almost certainly be empty as some other node will have found it and copied out all the bits. If, by some chance, it did find bits set in one or more bitmaps (as could happen when the first node joins), it will start a resync of its own.

#### Handling device failure

In some ways, handling device failure in the cluster is similar to handling device failure in a single node: the faulty device is recorded in the array metadata as “Faulty” and service continues on the remaining device(s). It is a little different in that all nodes need to acknowledge the failure, but since we have a message broadcast mechanism for that, it is quite straightforward.

One important question introduced by the presence of the cluster is: “Is it the device that is faulty, or the node?”. If a node has lost contact with a device, then it could just be that a cable has been disconnected. There are two credible responses to this situation. One is to tell all nodes to stop using the device, because this node cannot perform updates there anymore. The other is to decide that losing one node is better than losing redundancy in the array, so this node should leave the cluster and stop functioning.

This is a policy decision that MD is not in a position to make, yet it must make some decision. If the array still has redundancy (i.e. there are 2 or more mirrored devices), the best course is to tell all nodes that the device is faulty. This is the easiest way to provide continued service. If some cluster-wide monitoring software can determine that only one node is having problems, and that the device is really fine, it could disable the faulty node and reactivate the missing device.

While the array is degraded, bits in the bitmap of out-of-sync ranges are never cleared. This means that when a recently removed device is re-introduced to the array, it can be fully re-integrated by just updating all of the regions listed in the bitmap rather than recovering the entire device. This means that reactivating that missing device will normally happen fairly quickly. 

So if MD ejects a device due to an I/O error and the cluster-policy software decides this was the wrong choice, it can restore a fully redundant array with relatively little effort. Most of this would be controlled through udev scripts and such transient errors will usually be taken care of automatically.

If, on the other hand, the array is already degraded and MD detects a failure in the last remaining device, the options are different. In the single-node case, MD just passes the error up to the filesystem and lets it deal with it. In a cluster this is probably not ideal. The best solution in a cluster is probably to eject the node that has the problem, in the hope that other nodes can keep working. The best MD can do is to block the I/O request that caused the error and freeze the node. This leaves policy up to some other code. Cluster-health-management software can then either kill the node completely or instruct it to report the error to the filesystem.

#### Adding a new spare device

The one remaining task that is a little more interesting in a cluster than in a single node is integrating a new spare into the array. Related tasks, such as marking a device as failed or marking a specific spare as active, all involve changes to the set of devices that all nodes already know about. These changes can be achieved by updating the metadata and then asking all nodes to re-read and assess the new metadata — though in some cases clustered MD supports a more direct approach.

Adding a new device cannot be done with a simple “do this to that device”-style command because there is no a-priori guarantee that all nodes even know about the new device. MD uses a two-stage approach that involves support from user space and is open to the possibility of failure.

When `mdadm` is used to add a spare to a clustered array, MD will extract the (randomly assigned) device UUID from the MD superblock and broadcast a message to all nodes in the cluster asking them to add this device too. As they each get the message they trigger a “uevent” that will cause `udev` to run `mdadm` asking it to find that device. `mdadm` either adds the device to the array, or tells MD that the device cannot be found. MD then acknowledges the “add new disk” broadcast message.

The broadcast protocol that MD uses does not allow other nodes to send any negative acknowledgment. They just release a shared lock once they have handled the request so no success/failure can be passed back. This requires an extra piece of infrastructure.

When a node handles that “add new disk” message, it signifies success or failure either by releasing any lock that it might have on the resource called “no-new-dev”, or by taking a shared lock — respectively. When the originating node gets an acknowledgment for its broadcast it tries to get an exclusive lock on that resource. If this succeeds, then all other nodes must have successfully added the device. If it fails, then at least one node hasn't found the device.

After the “add” process completes — whether successfully or not — the on-disk metadata is updated and a message is broadcast to all nodes so they will re-read it. If the device-add failed because some node couldn't find the device, the new device will not be mentioned in the metadata, so those nodes that did successfully add it will know to remove it again.

#### Current and future status

The code currently in Linux 4.5-rcX implements nearly all of this, with a few remaining details to be finalized for 4.6. There are bound to be bugs but ongoing testing should resolve those soon enough.

A fairly obvious question once this is working reliably is: what about other RAID levels? The non-redundant levels of RAID 0 and Linear are already handled perfectly adequately by CLVM, so there is no value in adding support for those to MD. RAID 10 would not be much more complex than RAID 1, so if there was a genuine need and someone willing to pursue such an implementation, it would probably progress quite easily.

The striped/parity levels (RAID 4, RAID 5, and RAID 6) are more complex. As was mentioned at the start, the RAID 1 implementation depends on the fact that the filesystem won't write to the same block from two different nodes at the same time. For RAID-5-style levels the guarantee required would be that two different nodes don't write to the same stripe at the same time. As stripes are a lot bigger than blocks, that is much less likely.

Without this sort of guarantee from the filesystem, the only approach that might be workable would be for some node to be designated the primary node and for all others to send non-trivial requests over the network to that node. Trivial requests including reads from a working device and full-stripe writes could still be handled locally; that may provide improved performance over sending all I/O requests over the network.

While [some interest](<http://www.spinics.net/lists/raid/msg51011.html>) has been shown in cluster-support for RAID 5 there are as yet no concrete plans to implement anything. For now, RAID 1 is all we have in our sights.
