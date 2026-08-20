---
title: "A day in the life of linux-next"
url: https://lwn.net/Articles/287155/
date: "June 23, 2008"
category: "Development model-linux-next; linux-next"
author: "By Jonathan Corbet June 23, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 23, 2008

The merge window phase of the kernel development cycle is a hectic time. Over a period of about two weeks, between 5,000 and 10,000 changesets find their way into the mainline git repository. Simply managing that many patches would be hard enough, but the job is made more complicated by the fact that these changesets are not all independent of each other. The first changes to be merged can change the code base in ways that cause later patches to fail to apply. So merge windows have traditionally required maintainers to rework their queued patches to resolve conflicts which arise as other trees are merged. Given the tight time constraints (patches which aren't ready when the merge window closes generally sit out until the next cycle starts), this integration process has been known to put a fair amount of pressure on subsystem maintainers. 

The other person feeling the stress was Andrew Morton; one of his many jobs was to bash subsystem trees together in his -mm releases. That took a lot of his time and didn't really solve the problem in the end; much of the work which shows up in -mm isn't necessarily intended for the next development cycle. The end result of all this is that each merge window brought together large amounts of code which had never been integrated before. 

Back in February, the linux-next tree was [announced](<http://lwn.net/Articles/269120/>) as a way to help ease some of these problems. We are now nearing the end of the first full development cycle to use linux-next, so it's worth taking a look to see how it is working out. 

The idea behind this tree is relatively simple. Linux-next maintainer Stephen Rothwell keeps a [list of trees](<http://git.kernel.org/?p=linux/kernel/git/sfr/linux-next.git;a=blob;f=Next/Trees;hb=HEAD>) (maintained with git or quilt) which are intended to be merged in the next development cycle. As of this writing, that list contains 95 trees, all full of patches aimed at 2.6.27. Once a day, Stephen goes through the process of applying these trees to the mainline, one at a time. With each merge, he looks for merge conflicts and build failures. The [original plan for linux-next](<http://lwn.net/Articles/268881/>) stated that trees causing conflicts or build failures would simply be dropped. In reality, so far, Stephen usually takes the time to figure out the problem; he'll then fix up or drop an individual patch to make everything fit again. 

When this process is done, he releases the result as the linux-next tree for the day. Others then grab it and perform build testing on it; some people even boot and run the daily linux-next releases. All this results in a steady stream of problem reports, small fixes, patches moving from one tree to another, and so on - various bits of integration work required to make all of the pieces fit together nicely. 

There is an interesting sort of implicit hierarchy in the ordering of the trees. Subsystem trees which are merged early in the process are less likely to run into conflicts than those which come later. When two trees do come into conflict, it's the owner of the later tree - the one which actually shows the conflict - who feels the most pressure to fix things up. The history so far, though, shows that there has been very little in the way of finger-pointing when conflicts arise, as they do almost every day. All of the developers understand that they are working on the same kernel, and they share a common interest in solving problems. 

[PULL QUOTE:  One aspect of this whole system remains untested, though: the movement of patches from linux-next into the mainline.  END QUOTE] So, thus far, linux-next appears to be functioning as intended. It is serving as an integration point for the next kernel and helping to get many of the merging problems out of the way ahead of time. One aspect of this whole system remains untested, though: the movement of patches from linux-next into the mainline. As things stand now, there is no automatic movement between the trees; instead, maintainers will send their pull requests directly to Linus as always. If Linus refuses to merge certain trees, or if he merges them in an order different from their ordering in linux-next, integration problems could return. In the end, it seems like linux-next will have to drive the final integration process more than is anticipated now, but it will probably take a few development cycles to figure out how to make it all work. 

Meanwhile, anybody who is interested in 2.6.27 can, to a great extent, run it now by grabbing linux-next. This tree has clarified one aspect of the development process: the 2-3 month "development cycle" run by Linus is, in fact, just the tip of the kernel development iceberg. It is the final integration and stabilization stage. Linux-next nearly doubles the length of the visible development cycle by assembling the next kernel long before Linus starts working on it. And even linux-next only comes into play toward the end of a patch's life. 

In the past, Linus has pointedly worked to avoid overlapping the development and stabilization phases of the development cycle. There was no development tree at all for almost a year while 2.4 was beaten into reasonable shape. This separation was maintained out of a simple fear that an open development tree would distract developers from the more important task of finding and fixing bugs in the current stable release. 

That separation is a thing of the past now; there are literally dozens of development trees which are open for business at all times. That can only be worrisome to those who are concerned about the quality of kernel releases; why should developers concern themselves with [2.6.26 bugs](<http://lwn.net/Articles/287055/>) when 2.6.27 is being assembled and 2.6.28 is already on the radar? Whether such concerns are valid is likely to be a matter of ongoing debate. 

Meanwhile, however, linux-next appears to have settled in as a long-term feature of the kernel development landscape. It is serving its purpose as a place to find and resolve integration problems; it has also had the effect of taking much of that integration work off of Andrew Morton's shoulders. And that, in turn, should free him to spend more time trying to get developers to fix all those bugs. 

(See [the linux-next wiki](<http://linux.f-seidel.de/linux-next/pmwiki/>) for more information on how to work with this tree).
