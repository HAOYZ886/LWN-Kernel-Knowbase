---
title: Sparse gets a maintainer
url: https://lwn.net/Articles/208312/
date: "November 8, 2006"
category: "Development tools-Sparse; sparse"
author: 
---

The "sparse" utility has long been one of Linux's best-kept secrets. It is a static analysis tool which can find a wide variety of bugs in the kernel code base; sparse is a useful tool, but it can be surprisingly hard to find. It has never had a web page, and almost no distributions package it. Interested users must, instead, track down the git tree or Dave Jones's [snapshot directory](<http://www.codemonkey.org.uk/projects/git-snapshots/sparse/>). 

Sparse was originally written by Linus Torvalds, but he has not done much with it for a while, and he recently [suggested](<https://lwn.net/Articles/208314/>) that somebody else should take it over: 

Anyway, I suspect it would be better if people didn't consider me the maintainer for sparse, simply because it does the things I really cared about, and as a result I'm not really very active. 

As a result of this discussion, sparse has a new maintainer: Josh Triplett. Josh started things off with [sparse 0.1](<http://lwn.net/Articles/208043/>), the first-ever sparse release with a version number. He has set up a new git tree for sparse, and, even, a [sparse web page](<http://kernel.org/pub/linux/kernel/people/josh/sparse/>). 

Josh was kind enough to answer some questions posed by your editor. It turns out that he has been working with sparse for a while; it was part of his PhD work, where he enhanced it to be able to verify proper use of the read-copy-update (RCU) primitives. That work continued at IBM over the summer, where he was able to work on RCU verification with Paul McKenney. 

As a result, his first priority for sparse in the near future is the continuation of the RCU work. This effort is also expanding into locking verification in general; some of the necessary annotations and resulting fixes have gone into the 2.6.18 and 2.6.19-rc kernels. Josh also plans to work on the elimination of false positives and on noise reduction in general. Then, there's various patches from other developers which have been floating around for a while and really need to be merged into the sparse mainline. 

In terms of project management, Josh says: 

I plan to continue making regular Sparse releases, and I'd like to get Sparse packaged in various distributions, at least in their "experimental" sections or equivalent. Any potential distribution packagers, feel free to join the linux-sparse list, and let me know what I can do to help or to get things going more smoothly. 

Getting sparse into distributions could only help increase its use - and bring about a corresponding reduction in bugs in shipped code. This will be especially true if Josh succeeds in another one of his goals: expanding sparse usage beyond the kernel into user-space projects. X.org seems like it could be an early sparse adopter. 

Longer-term, Josh wants to look at more advanced techniques which can look at larger chunks of a program and find potential bugs. Part of this effort will require attracting other researchers interested in static analysis to the sparse platform. Says Josh: 

I feel that several classes of bugs exist in the Linux kernel and in userspace code which simply should not exist, because the tools exist to find and eliminate almost all of them. This includes bugs like "scheduling while atomic", __init-related bugs, errors on error paths, and many locking-related bugs. 

One can only imagine that free software users all over are wishing Josh the best of luck in his effort to track down and get rid of all those unnecessary bugs.
