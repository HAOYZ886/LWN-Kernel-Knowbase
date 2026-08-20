---
title: NFS topics
url: https://lwn.net/Articles/788292/
date: "May 14, 2019"
category: "Filesystems-NFS"
author: "By Jake Edge May 14, 2019 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
May 14, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Trond Myklebust and Bruce Fields led a session on some topics of interest in the NFS world at the 2019 Linux Storage, Filesystem, and Memory-Management Summit. Myklebust discussed the intersection of NFS and containers, as well adding TLS support to NFS. Fields also had some container changes to discuss, along with a grab bag of other areas that need attention. 

Myklebust began with TLS support for the RPC layer that underlies NFS. One of the main issues is how to do the upcall from the RPC layer to a user-space daemon that would handle the TLS handshake. There is [kernel support for doing TLS](<https://lwn.net/Articles/666509/>) once the handshake is complete; hardware acceleration of TLS was added in the last year based on code from Intel and Mellanox, he said. RPC will use that code, but there is still the question of handling the handshake. 

[ ![\[Trond Myklebust\]](https://static.lwn.net/images/2019/lsf-myklebust-sm.jpg) ](<https://lwn.net/Articles/788295/>)

There are a few different options for the handshake. It could use the same kind of upcall that [`rpc.gssd`](<http://man7.org/linux/man-pages/man8/rpc.gssd.8.html>) does. That would require a daemon listening for handshake requests; once the handshake is complete it would hand the connection back to the kernel to use for the RPC traffic. 

Another option would be to do something based on [netlink](<http://man7.org/linux/man-pages/man7/netlink.7.html>). That would be more generic, but is more appropriate to discuss with the networking developers. He plans to determine if those developers are interested in using netlink for that and will fall back to using the `rpc.gssd` approach if not. He knows the latter works well with containers and the types of applications in use. 

For containers, he said that there are a fair number of patches going into the kernel recently; more are queued up in Fields's and Anna Schumaker's trees. The main issue that still needs to be dealt with is how to handle user namespaces, he said. There are two problems there, the first is that the NFSv4 ID mapper ([`rpc.idmapd`](<http://man7.org/linux/man-pages/man8/rpc.idmapd.8.html>)) has been using the keyring upcall interface, which does not support user namespaces at all. The kernel has not been able to map the kernel user ID (kuid) to the user ID inside the user namespace where `rpc.idmapd` is running. There are patches queued up to rectify that. 

The other issue is which IDs get put on the wire. When using NFSv3 and some configurations of NFSv4, raw user and group IDs are sent by clients to the server. Should those be the kernel user/group ID or those of the container? The plan is to use the IDs from inside the namespace, as there is "no real point" in hiding them or translating them to something other than what is seen in the container. 

Fields asked if there were other container gaps that Myklebust knew of. He said that there is still an issue with DNS lookups from within a container, because that also uses the keyring upcall interface. That should be fixed, but is not critical because it is not used for a lot of things. 

The NFSv4 state in different containers should be different so tenants on the same system don't share the same client ID, Fields said, and wondered if that problem had been solved. Myklebust said that many containers do not set a hostname, which makes it difficult to determine what ID to use to set up a lease when talking to the NFS server. He has looked into a way to create an ID in a generic way so that other filesystems could also use it rather than doing something in an NFS-specific daemon. 

Chuck Lever asked if he was referring to "clientid4" (which is part of the [NFSv4 protocol](<https://tools.ietf.org/html/rfc5661>)). Myklebust said that he was; he has something working using a udev upcall into the container namespace, but hasn't yet published the patches. It will also be used for non-containerized NFS clients because there are a number of Linux distributions that do not require setting a hostname. That leads to a lot of "localhost.localdomain" leases, which is not desirable. The patches will require some cleanup, so it will be Linux 5.3 or later before they will be upstream. 

At that point, Fields stepped up to the lectern. There is some amount of state that the server needs to store to track clients across server reboots so that clients can reclaim their locks, he said. The way that was being done did not work well for containers, but that has been fixed, though there are both kernel and user-space parts, so it may take a little while for it all to roll out. 

[ ![\[Bruce Fields\]](https://static.lwn.net/images/2019/lsf-fields-sm.jpg) ](<https://lwn.net/Articles/788296/>)

He is concerned that the duplicate reply cache is global to the server, which means that it is shared with all containers. He is fairly certain that malicious clients could snoop the cache, but it could also be a source of bugs since clients could get the wrong cache entry. Either creating separate caches for each client or keying the cache entries using the network namespace should take care of that problem. 

Lever asked about containerizing the performance metrics for the server. That does need to be looked at, Fields said. Some of the metrics should be global, but others may not. What is needed is for more people to be using NFS from containers because there may be more corner cases that need to be handled, he said. 

Server-to-server copy offload is something that is being worked on. When those patches were posted, Dave Chinner noticed some problems in the filesystem and VFS layers; Chinner sent out patches to fix those problems, but has not pushed them further, possibly due to lack of time. It requires someone picking them up and getting them upstream, Fields said. 

Next up was delegations, which are a mechanism where the server can grant exclusive read or write access to a client for a file. Multiple clients can have read delegation for a file, but if another client opens the file for write the server needs to revoke any read delegations. But if there is only one client that has a read delegation and it is the one that writes to the file, there is no need to revoke the delegation. 

In order to implement that, there is a need to track the client ID all the way through the VFS. Trying to plumb that through all of the VFS was not realistic. The second attempt, which was to put something in the task structure, was not popular with other developers. The latest attempt is to use the thread-group ID (tgid). In order to do that, he had to make some small modifications to the kernel thread daemon (kthreadd) so that the NFS daemon could run a private version to get all of its threads into the same thread group. Nobody seemed to have objections to that approach, but he is not sure who is supposed to review code in that area. 

Steve French asked about adding file attributes, such as those now available using the [`statx()`](<http://man7.org/linux/man-pages/man2/statx.2.html>) system call, to the protocol. There are some attributes that are being considered as part of the internet draft, such as the archive bit, Lever said. Others, like birth time, have already been added to the specification, Myklebust said, so they could be added to the Linux NFS implementation. 

Ted Ts'o referred to the [support for case insensitivity](<https://lwn.net/Articles/784041/>) that has recently been added for ext4. He asked if there is anything that needs to be done so that NFS can also use it. Myklebust said there is work needed on both the NFS client and server in order to support that. He is waiting to see what lands in the kernel (the feature was [merged for Linux 5.2](<https://lwn.net/Articles/787963/>) after LSFMM) before looking into all that needs to be done. He said that there will at least be changes needed for file name lookups and in managing the directory entry cache (dcache).
