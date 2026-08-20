---
title: Asynchronous buffered read operations
url: https://lwn.net/Articles/636967/
date: "March 18, 2015"
category: Asynchronous IO
author: "By Jake Edge March 18, 2015 LSFMM 2015"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
March 18, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

A problem that Milosz Tanski has run into throughout his career is part of what brought him to the 2015 Linux Storage, Filesystem, and Memory Management Summit. Some reads can be satisfied immediately from the page cache, while others require an expensive I/O. Distinguishing between the two can lead to more efficient programs. He has implemented a new mode for `read()` that does so, though it requires adding a new system call. 

The problem typically occurs in low-level network applications, Tanski said. Not every application can use `sendfile()`. For example, applications using TLS modify the data to encrypt it before sending it, which means they can't use `sendfile()`. So they must do their own copies but, depending on whether the data is in the page cache, some will be "slow", while others are "fast". Some programs that want to do asynchronous disk I/O often just use `O_DIRECT` and replicate the page cache concept in user space. That way they can track the contents of the cache to determine if an I/O can be satisfied quickly or not. 

[ ![\[Milosz Tanski\]](https://static.lwn.net/images/2015/lsf-tanski-sm.jpg) ](<https://lwn.net/Articles/636979/>)

The normal workaround for these problems is to use thread pools for the I/O, but that pattern "kinda sucks". The latency added due to synchronization between the threads is not insubstantial. It is also often the case that requests that could be satisfied quickly get stuck behind slower requests. 

So, with the help of Christoph Hellwig, he has [implemented `preadv2()`](<https://lwn.net/Articles/612483/>), which is like `preadv()` except that there is a new `flags` argument (which, as was pointed out by several attendees, really [should have been added](<https://lwn.net/Articles/585415/>) with `preadv()`). There is only one flag available in his [patches](<https://lwn.net/Articles/636955/>): `RWF_NONBLOCK` (which could also have been called `RWF_NOWAIT`, he said). That flag will cause reads to succeed only if the data is already in the page cache, otherwise it will return `EAGAIN`. 

Basically, that flag allows reads from the network loop to skip the queue if the data needed is already available in the page cache. It essentially provides a fast path with minimal changes to the user-space application. He has been using it with an internal application and it works well. 

His patches drew one major comment, he said, which was about using functionality like that in [`fincore()`](<https://lwn.net/Articles/371538/>) to get a list of the pages of a file that are resident in the page cache. The problem with that is a race condition where a page that was present at the time of the check is no longer there when the read is performed, which puts that read back into the slow lane. 

He has also tested the patches with Samba, where they reduce the latency significantly. For his internal application, which is a large, columnar data store using the Ceph filesystem, he got 23% lower response times. The average response times dropped by 200ms, he said. 

There have been some objections to adding another system call, Tanski said. James Bottomley was not particularly concerned about that, since the new system call is just adding a flag argument that should have been there already. Hellwig added that it required a new system call just to get the flag in, which is not an unusual situation in recent times. 

Hellwig has also implemented `pwritev2()` as part of the patch set to add a flag argument for the `write()` side. There are no write flags included in the patch, though some will be added as separate patches down the road. There are some potential user-space uses for flags for writes, including a "high priority" flag and a non-blocking flag that could be used for logging, Hellwig said. 

No one in the room seemed opposed to the idea. It seems likely that the two new system calls could show up as early as the 4.1 kernel. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
