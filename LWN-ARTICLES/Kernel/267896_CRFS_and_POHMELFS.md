---
title: CRFS and POHMELFS
url: https://lwn.net/Articles/267896/
date: "February 6, 2008"
category: "Filesystems-Network; Network filesystems"
author: "By Jake Edge February 6, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
February 6, 2008

Performance, or lack thereof, has often been a knock against the venerable Network File System (NFS), but no real competition has emerged. NFS also has some serious flaws for programmers and users, with behavior that is markedly different from that of local filesystems. Both of these problems are spurring the creation of new network filesystems; two of which were announced in the last week. 

The Coherent Remote File System (CRFS) was introduced last week at linux.conf.au by Zach Brown of Oracle. It uses [BTRFS](<http://lwn.net/Articles/265257/>)--pronounced "butter-f-s"--as its storage on the server, rather than layering atop any POSIX filesystem as NFS does. According to Brown, BTRFS has a number of important features that outweigh the inconvenience for users of getting their data into a BTRFS volume. The biggest is the ability to do compound operations (creating or unlinking a file for example) in an atomic and idempotent manner. 

CRFS has a userspace daemon (crfsd) that talks to the BTRFS volume as well as multiple clients. The clients use the kernel VFS caching infrastructure extensively, thus are implemented as kernel modules. A user wishing to access the underlying BTRFS volume on the server, must mount it as a CRFS volume; crfsd must have exclusive access to the BTRFS. This is also different from NFS which will cooperate with local mounts of the underlying filesystem. 

The basic idea behind CRFS is to have clients cache as much of the filesystem data as they can while using cache coherency protocols to reduce the amount of network traffic that gets generated. Clients keep track of the cache state for each object they have stored, while the server tracks the cache state of all objects that any client has. The messages between server and client consist of cache state transitions and the data being transferred. 

Data transfer in both directions is done using CRFS "item ranges". CRFS objects use the BTRFS key scheme to represent objects (file data, directories, directory entries, inodes, etc.) in the filesystem. An item range is a contiguous section of the key space, specified by a minimum and maximum key value as part of the message. When the client is filling its cache, it can request a particular key but also offer to take other surrounding keys as part of the response; if the server sees those keys in the BTRFS leaf node, it can send them along as well. 

Something on the order of a 3x speedup over asynchronous NFS mounts is the current performance of CRFS for a simple untar. Comparing to synchronous NFS mounts (where each write has to actually hit the remote disk) is not a sensible comparison; there is a roughly 10x speed difference between the two types of NFS mounts. Brown has been working on CRFS for "about a year" and is planning to release the code eventually. Until that happens, the [slides [PDF]](<http://mirror.linux.org.au/pub/linux.conf.au/2008/slides/247-CRFS-lca2008-slides.pdf>) and [video [Theora]](<http://mirror.linux.org.au/pub/linux.conf.au/2008/Fri/mel8-247.ogg>) from his talk--as well as a few [postings to his weblog](<http://www.zabbo.net/>)--are the only sources of information about CRFS. 

Another filesystem, that aims to have a broader reach than CRFS, is the [Parallel Optimized Host Message Exchange Layered File System](<http://lwn.net/Articles/267790/>) (POHMELFS), announced in linux-kernel posting by Evgeniy Polyakov. POHMELFS is meant to be a building block for a distributed filesystem that would offer a multi-server architecture and allow for disconnected filesystem operations. Polyakov has only been working on it for a month, so it is, at best, the start of a proof of concept. 

The POHMELFS vision is in some ways similar to CRFS in that the clients will handle as much as possible locally, with minimal server interaction. Like CRFS, client kernel modules talk to a server userspace daemon, using cache coherency protocols to keep the data and metadata in sync. For CRFS, the coherency is not yet implemented, but is fleshed out to some extent, while POHMELFS has quite a bit of fleshing out to do. Unlike CRFS, POHMELFS supports POSIX filesystems on the server side and the code is available now. 

There are some rather large hurdles to overcome in the POHMELFS vision, not least of which is handling file IDs in separate client-side filesystems such that they can be synchronized with the server. The current code implements a write-through cache version that creates objects on the server before they are used in the client side cache. There is also an additional patch that implements a hack to disable the writeback cache and use only the client side caching. The latter is, not surprisingly, very fast, but not terribly usable for multiple mounts of the filesystem. Essentially Polyakov is showing the benefits of client-side caching, but in the context of a broader scheme. 

It will be a long time, if ever, that we see some descendant of either of these filesystems in the kernel. There is much work to be done, but they are worth looking at to see where networking and distributed filesystems may be headed. For them to be useful outside of just the Linux world--like the ubiquity of NFS--there would have to be some kind of standardization followed by adoption by the major players. That will take a very long time.
