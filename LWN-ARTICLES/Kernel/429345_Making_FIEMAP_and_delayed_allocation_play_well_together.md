---
title: Making FIEMAP and delayed allocation play well together
url: https://lwn.net/Articles/429345/
date: "February 22, 2011"
category: FIEMAP ioctl
author: "By Jonathan Corbet February 22, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 22, 2011

The `FIEMAP` `ioctl()` command can be used to learn about how a file's blocks are laid out on the disk. It's useful for determining fragmentation, optimizing boot-time readahead order, and a number of other things. One of those other things, though, has turned up bugs in how a couple of important filesystems implement `FIEMAP`. 

The `cp` application, it seems, has recently been taught to use `FIEMAP` to find holes in files. The idea is to optimize the copying of such files by not even reading the holes; that way, the need to zero-fill pages (in the kernel) and compare them against pages full of zeros (in user space) can be eliminated. It seems like a better way of doing things. 

Somewhere along the way, Chris Mason got word that `cp` was corrupting files on btrfs filesystems. The problem, naturally enough, was that `FIEMAP` was reporting holes where none should exist. The root cause was that `FIEMAP` was not prepared to deal with regions of a file which have been written to, but which do not actually have blocks assigned yet. The delayed allocation mechanism used by most contemporary filesystems will create exactly that kind of situation, so this is not a theoretical concern. 

Chris [fixed the problem](<https://lwn.net/Articles/429347/>) for btrfs, then decided to see how other filesystems handled the same situation. From [his report](<https://lwn.net/Articles/429349/>), xfs handled things well, but ext4 had similar bugs in situations where delayed allocation and real holes came together in the same file. Certain types of bugs, it seems, are likely to turn up in more than one context. 

Chris's fix should get into 2.6.38 before the final release; chances are good that an ext4 fix will be fast-tracked as well. Expect stable kernel backports too. In the meantime, be careful when copying recently-written files with new versions of `cp` on those filesystems.
