---
title: Tux3 posted for review
url: https://lwn.net/Articles/599795/
date: "May 21, 2014"
category: "Filesystems-Tux3"
author: "By Jonathan Corbet May 21, 2014"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 21, 2014

After years of development and some seeming false starts, the Tux3 filesystem has been [posted for review](<https://lwn.net/Articles/599348/>) with the hope of getting it into the mainline in the near future. Tux3, [covered here](<https://lwn.net/Articles/309094/>) in 2008, promises a number of interesting next-generation filesystem features combined with a high level of reliability. This posting is a step forward for Tux3, but it will still probably be some time before it finds its way into the mainline. 

The only developer to review the code so far is Dave Chinner, and he [was not entirely impressed](<https://lwn.net/Articles/599797/>). There is a lot of stuff to clean up, but Dave is most concerned about various core memory management and filesystem changes that, he says, need to be separated out for review on their own merits. One of the core Tux3 mechanisms, called "page forking," [was not well received](<https://lwn.net/Articles/548091/>) at the 2013 Storage, Filesystem and Memory Management Summit, and Tux3 developer Daniel Phillips has done little since then to address the criticisms heard there. 

Dave is also worried about the "work in progress" nature of a number of promised Tux3 features. Years ago, Btrfs was merged while in an incomplete state in the hope of accelerating development; Dave now [says](<https://lwn.net/Articles/599799/>) that was a mistake he does not want to repeat: 

The development of btrfs has shown that moving prototype filesystems into the main kernel tree does not lead stability, performance or production readiness any faster than if they stayed as an out-of-tree module until most of the development was complete. If anything, merging into mainline reduces the speed at which a filesystem can be brought to being feature complete and production ready. 

All told, it adds up to a chilly reception for this new filesystem. Daniel appears to be up to the challenge of getting this code into shape for merging, though. If he follows through, we should start seeing smaller patch sets that will facilitate the review of specific Tux3-related changes. Only after that process completes will it be time to look at getting the filesystem itself into the mainline.
