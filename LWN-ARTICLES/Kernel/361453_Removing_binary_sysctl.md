---
title: Removing binary sysctl
url: https://lwn.net/Articles/361453/
date: "November 11, 2009"
category: "Development model-User-space ABI; Sysctl"
author: "By Jonathan Corbet November 11, 2009"
---

By **Jonathan Corbet**  
November 11, 2009

The "sysctl" mechanism is used by the kernel to export a wide variety of tuning options to user space. Sysctl is actually two interfaces which have been awkwardly joined together: the `sysctl()` system call and the `/proc/sys` directory hierarchy. Of the two, `/proc/sys` is much more widely used, to the point that developers rarely even think about the system call. But the `sysctl()` implementation is a significant amount of code which suffers from chronic neglect. It has thus been [the target of removal attempts](<http://lwn.net/Articles/204935/>) for years. 

The problem with removing `sysctl()`, of course, is that it is part of the kernel ABI. As long as the possibility of broken applications exists, this ABI cannot be removed. So it continues to sit in the kernel, despite the fact that its absence would be noted by few people. 

Eric Biederman has come up with [a new approach to the problem](<http://lwn.net/Articles/361001/>). His patch set removes the current `sysctl()` implementation, getting rid of a few thousand lines of unloved code. He then adds back a new wrapper which emulates the `sysctl()` ABI by way of `/proc/sys`. So any applications using `sysctl()` should continue to work, but the code dedicated to making it work is much reduced from what was there before. 

The patch set still concerns some developers. The compatibility wrapper has its own configuration option, leading some to worry that distributions might disable it and cause obscure things to break. Going through `/proc/sys` will make access to these variables much slower than it was before. That should not really be a problem: access to sysctl variables is not normally a performance-critical operation. So there does not appear to be any sort of real obstacle to the merging of these patches; maybe, someday, binary `sysctl()` will truly vanish into the past.
