---
title: Trustees Linux
url: https://lwn.net/Articles/111247/
date: "November 16, 2004"
category: "Capabilities; Security-Security modules"
author: 
---

Linux currently offers a wealth of projects which are working to extend the classic Unix permissions mechanism with more flexible schemes. One recent entry is [an LSM port of Trustees Linux](<https://lwn.net/Articles/110986/>), which has been done by Andrew Ruder. Trustees Linux starts with the idea that access control lists are overly complicated and inefficient; achieving the desired goals can require hanging ACLs on thousands of files, and keeping all of those ACLs in sync can be a challenge. 

The Trustees approach, instead, is to create a separate, central database which contains filesystem permissions. This database can assign a "trustee" to a directory; this trustee provides access permissions which apply to the directory and, by default, everything below that directory. A single rule can, thus, cover a large part of the filesystem hierarchy. 

The trustee rules cover the usual sorts of permissions; who can search for, read, and write files in a given subtree. The format is somewhat terse; one of the rules provided in the examples is: 

```
[/dev/hda1]/var/log:zavadsky:REB
```

This rule enables user "zavadsky" to wander around in (and under) `/var/log` and read files there. 

Mr. Ruder's port is centered around the Linux security module `inode_permission()` hook; that code examines the trustees which apply to a given inode and decides whether the requested access is to be allowed or not. 

It's all pretty straightforward, but there is an interesting twist to how Trustees works with file permissions: the module gives the `CAP_DAC_OVERRIDE` capability to every process, allowing them to override the existing Unix file permissions. The Trustees module will, in turn, apply those permissions itself much of the time, but it is possible to write rules which override them. In this sense, Trustees functions as an authoritative module, which is not how LSM modules are supposed to work. If Trustees Linux is ever proposed for merging into the mainline, that little feature could come back to haunt it.
