---
title: "A new kernel tree: linux-staging"
url: https://lwn.net/Articles/285594/
date: "June 11, 2008"
category: Staging tree
author: "By Jake Edge June 11, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
June 11, 2008

There's a new kernel tree in town. The linux-staging tree was [announced](<http://lwn.net/Articles/285599/>) by Greg Kroah-Hartman on 10 June. It is meant to hold drivers and other kernel patches that are working their way toward the mainline, but still have a ways to go. The intention is to collect them all together in one tree to make access and testing easier for interested developers. 

According to Kroah-Hartman, linux-staging (or -staging as it will undoubtedly be known) ""is an outgrowth of the Linux Driver Project, and the fact that there have been some complaints that there is no place for individual drivers to sit while they get cleaned up and into the proper shape for merging."" By collecting the patches in one place, it will increase their visibility in the kernel community, potentially attracting more developers to assist in fixing, reviewing, and testing them. 

The intent is for -staging to house self-contained patches--Kroah-Hartman mentions drivers and filesystems--that should not affect anyone who is not using them. Because of that, he is hoping that -staging can get included in the [linux-next](<http://lwn.net/Articles/269120/>) tree. As he says to Stephen Rothwell, maintainer of -next, in the announcement: 

Yes, I know it contains things that will not be included in the next release, but the inclusion and basic build testing that is provided by your tree is invaluable. You can place it at the end, and if there is even a whiff of a problem in any of the patches, you have my full permission to drop them on the floor and run away screaming (and let me know please, so I can fix it up.) 

The -next tree is meant for things that are headed for inclusion in the "N+1" kernel (where 2.6.N is the release under development), so including code not meant for that release is bending the rules a bit. As of this writing, Rothwell has not responded to the request to include -staging, but it would clearly benefit those patches to have a wider audience--with only a small impact on -next. There is no set timeline for patches to move from -staging into mainline, Kroah-Hartman says: 

Based on some of the work that is needed on some of these drivers, it is much longer than N+2, unless we have some people step up to help out with the work. It's almost all janitorial work to do, but I know I personally don't have enough time to do it all, and can use the help. 

The -staging tree is seen as a great place for Kernel Janitors and others who are interested in learning about kernel development to get their start. The announcement notes: ""The code in this tree is in desperate need of cleanups and fixes that can be trivially found using 'sparse' and 'scripts/checkpatch.pl'."" In the process of cleaning up the code, folks can learn how to create patches and how to get them accepted into a tree. From there, the hope is that more difficult tasks will be undertaken--with -staging or other kernel code--leading to a new crop of kernel hackers. 

The current [status of -staging](<https://lwn.net/Articles/285600/>) shows 17 patches, most of which are drivers from the Linux Driver Project. Kroah-Hartman is actively encouraging more code to be submitted for -staging, as long as it meets some criteria for the tree. The tree is _not_ meant to be a dumping ground for drivers that are being "thrown over the wall" in hopes that someone else will deal with them. It is also not meant for code that is being actively worked on by a group of developers in another tree somewhere--the reiser4 filesystem is mentioned as an example--it is for code that would otherwise languish. 

The reaction on linux-kernel has so far been favorable, with questions being asked about what kinds of patches are appropriate for the tree, in particular [new architectures](<https://lwn.net/Articles/285678/>). The -staging tree fills a niche that has not yet been covered by other trees. It also serves multiple purposes, from giving new developers a starting point to providing additional reviewing and testing opportunities for new drivers and other code. With luck, that will hasten the arrival of new features--along with new developers.
