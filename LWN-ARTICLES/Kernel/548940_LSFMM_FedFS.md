---
title: "LSFMM: FedFS"
url: https://lwn.net/Articles/548940/
date: "May 1, 2013"
category: "Filesystems-Federated filesystem"
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

A short session on the [Federated Filesystem](<http://tools.ietf.org/html/rfc5716>) (FedFS) was led by Chuck Lever at the 2013 LSFMM Summit. He mostly wanted to try to reach a consensus on how to store redirection information for particular files in FedFS. 

Lever started with a brief overview of FedFS, which aggregates multiple network filesystems into a single namespace. The idea is that an administrator could set up multiple NFS and Samba servers that all appear to a client to be in a single filesystem namespace. 

In order to handle filesystems that move to a different server or location, FedFS uses objects called "junctions" that contain information about where the filesystem has gone. The client can read that information and redirect requests to the proper location. FedFS uses extended attributes (xattrs) on a directory to store that information. Lever said that the current implementation uses XML, though attendees suggested using JSON instead. 

Samba has a different representation for the junction information. It stores it as the target of a symbolic link. Jeremy Allison suggested sticking with symbolic links as any new inode type for a junction would be Linux-specific. In the end, it was agreed that FedFS could use both the xattrs and the target of a symbolic link, so that both Samba and FedFS would be able to use the information.
