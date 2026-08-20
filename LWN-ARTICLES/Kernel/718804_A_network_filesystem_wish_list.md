---
title: A network filesystem wish list
url: https://lwn.net/Articles/718804/
date: "April 5, 2017"
category: "Filesystems-Network"
author: "By Jake Edge April 5, 2017 LSFMM 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
April 5, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

In the filesystem track and the 2017 Linux Storage, Filesystem, and Memory-Management Summit, Steve French led a discussion on various topics of interest for network filesystems. As with the [discussion at LSFMM 2016](<https://lwn.net/Articles/685431/>), there are a number of features that the network filesystem developers would like to see added, though there has been progress on one of primary items from last year: the `statx()` system call. 

[ ![\[Steve French\]](https://static.lwn.net/images/2017/lsfmm-french-sm.jpg) ](<https://lwn.net/Articles/718797/>)

French said that the addition of `statx()` gave Samba and other network filesystems some things that were wanted for a long time. The new system call adds a "birth time" for files, as well as two of the Windows attribute flags. But more is needed. Additional attribute flags, an interface to set attributes, and something like the [`READDIRPLUS` NFS command](<https://tools.ietf.org/html/rfc1813#section-3.3.17>) are all on that list. 

Samba leases, which allow the client side to aggressively cache file data, are not fully supported. The API lacks a way to provide a lease key in order to upgrade an existing lease, so upgrading requires dropping the lease, which is inefficient. There is also no way to cache metadata and directory contents, which Microsoft says can result in an enormous reduction in network traffic for typical home-directory-oriented users. 

French also noted that version 28 of the rich access-control list (RichACLs) had recently been [posted](<https://lwn.net/Articles/714386/>). Some small pieces of the patch set were merged over the last year, but there is a need for full RichACL support. Right now, NFSv4 and Samba ACLs are mapped as best as they can be using extended attributes (xattrs). But not all filesystems store xattrs efficiently and he is also worried that the mapping is imperfect, which could lead to security problems. 

There are races when creating files with ACLs and other attributes right now, French said. Jeff Layton thought that using `O_TMPFILE`, setting the ACLs and attributes on that file, then moving the file to its real name should be sufficient to avoid those races. French said he did not have the details but, from what he understands, the races are unavoidable with the current interfaces. There is a surprising amount of work needed at file creation time, he said. 

Network filesystems need broader support for the fast copy options (e.g. server-side copying and copy offload). Almost everyone wants their copies to be done quickly by default, but there is no Linux interface to simply hand a source and target to and have it make the best effort to do that copying quickly. There is a need for a per-file snapshot interface. Right now, there are filesystem-specific ways to request snapshots, but it would be helpful to have a single interface for all filesystems that can support it. Windows has that support, he said. 

There is currently no interface to get metadata about the filesystem itself. There are things that XFS and Btrfs know that could help client applications make better decisions. The alignment of the device, whether there is a seek penalty, or if the `TRIM` command is supported are all things that would be helpful to know. He noted David Howells's proposal for filesystem query system call (possibly `statfsx()`) that was made in the [`statx()` session](<https://lwn.net/Articles/718222/>); the timestamp granularity example given there was a good one, French said. 

Ted Ts'o said that most would probably agree that many of the interfaces French is talking about would be good additions, but they are also easy to bikeshed over so it "takes forever to get them upstream". The filesystem information system call is definitely needed and there is no need to convince the kernel developers of that, but there will be three months to three years of bikeshedding over it. The idea behind `statx()` was not controversial, Ts'o said, it's just that the details took a long time to be worked out. 

But French said that `statx()` is an example of progress. It was finally pared down to just adding birth time and two attributes, which is great, but it is an extensible interface. The "compressed" and "encrypted" attributes were added with the new call, but more would be helpful and he is optimistic that they will be added.
