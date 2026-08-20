---
title: A revoke() update and more
url: https://lwn.net/Articles/592116/
date: "April 2, 2014"
category: revoke
author: "By Jake Edge April 2, 2014 2014 LSFMM Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
April 2, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

Al Viro gave an update on the long-awaited `revoke()` system call to the 2014 Linux Storage, Filesystem, and Memory Management (LSFMM) Summit. `revoke()` is meant to `close()` any existing file descriptors open for a given pathname so that a process can know that it has exclusive use of the file or device in question. Viro also discussed some work he has been doing to unify the multiple variants of the `read()` and `write()` system calls. 

[ ![\[Al Viro\]](https://static.lwn.net/images/2014/lsfmm-viro-sm.jpg) ](<https://lwn.net/Articles/592787/>)

Viro started out by saying that `revoke()` was the less interesting part of his session. It is getting "more or less close to done", he said. We [looked](<https://lwn.net/Articles/546537/>) at an earlier version of this work a year ago. Files will be able to be declared revokable at `open()` time. If they are, a counter will track the usage of the `file_operations` functions at any given time. Once `revoke()` is called, it waits for all currently active threads to exit the `file_operations`, and makes sure that no more are allowed to start. 

There are places in procfs and sysfs where something similar is open-coded, Viro said, that could be removed once the `revoke()` changes go in. One of the keys is to ensure that the common path does not slow down for `revoke()` since most files will not be revokable. There are several areas that still need work, including `poll()`, which "provides some complications", and `mmap()`, which has always been problematic for `revoke()`. 

In a bit of an aside, Viro noted that there is a lot of code that is "just plain broke". For example, if a file in debugfs is opened and the underlying code removes the file from the debugfs directory, any read or write operation using the open file descriptor will oops the kernel. Dynamic debugfs is completely broken, Viro said. He hopes that the `revoke()` code will be in reasonable shape in a couple of cycles—"it's getting there". Dynamic debugfs will be one of the first users, he said. 

Viro then moved on to the unification of plain `read()` and `write()` with the `readv()`/`writev()` variants as well as `splice_read()` and `splice_write()`. The regular and vector variants (`readv()`/`writev()`) have mostly been combined, he said. It is "not pretty", but it is tolerable. The splice variants got "really messy". 

Ideally, the code for all of the variants should look the same all the way down, until you get to the final disposition. But each of the variants has its own view of the data; the splice variants get/put their data into pages, which doesn't fit well with the `iovec` used by the other two variants (in most implementations, plain `read()` and `write()` are translated to an `iovec` of length one). Creating a new data structure that can hold both user and kernel `iovec` members, along with `struct page` for the splice variants may be the way to go, Viro said. 

Something that "fell out" of his work in this area is the addition of `iov_iter`. The `iov_shorten()` operation tries to recalculate the number of network segments that fall into a given `iovec` area, but the result is that the `iovec` gets modified when there are short reads or writes. Worse still, how the `iovec` gets modified is protocol-dependent, which makes it hard for users. In fact, someone from the CIFS team said that it makes a copy of any `iovec` before passing it in because it doesn't know what it will get back. 

Having it be protocol-dependent is "just wrong", Viro said. He has been getting rid of `iov_shorten()` calls, as well as other places that shorten `iovec` arrays. That might allow `sendpage()` to be removed entirely; protocols that want to be smart can set up an `iov_iter`, he said. 

[ Thanks to the Linux Foundation for travel support to attend LSFMM. ]
