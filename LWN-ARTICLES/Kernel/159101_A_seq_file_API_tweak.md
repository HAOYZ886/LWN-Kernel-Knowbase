---
title: A seq_file API tweak
url: https://lwn.net/Articles/159101/
date: "November 8, 2005"
category: seq file
author: 
---

The [seq_file mechanism](<http://lwn.net/Articles/22355/>) is a helper for kernel subsystems wanting to create lengthy virtual files, usually in `/proc`. 2.6.15 will include a small enhancement which may prove helpful for some users. 

When user space opens a virtual file, the kernel must, in turn, call `seq_open()` to set things up. On return, the `file` structure passed to `seq_open()` will have, in its `private_data` field, a pointer to the `seq_file` structure created at open time. That is the same structure which will be passed to the seq_file iterator functions, and which must be used when actually generating output. 

Traditionally, `seq_open()` has always allocated the `seq_file` structure itself. In 2.6.15, however, it will examine the `private_data` field first, and, if that field is non-`NULL`, it will assume that the `seq_file` has already been allocated by the caller. This change allows seq_file users to embed the structure within something larger. It is worth noting, though, that `seq_release()` still frees the `seq_file` structure regardless of who created it. Among other things, that implies that, if the caller allocates a `seq_file` structure within a larger structure, the `seq_file` structure must appear at the beginning.
