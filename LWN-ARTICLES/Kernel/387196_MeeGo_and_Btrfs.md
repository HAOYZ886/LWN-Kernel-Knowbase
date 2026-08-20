---
title: MeeGo and Btrfs
url: https://lwn.net/Articles/387196/
date: "May 11, 2010"
category: "Filesystems-Btrfs"
author: "By Jonathan Corbet May 11, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 11, 2010

MeeGo is arguably the dark horse in the mobile platform race: it is new, unfinished, and unavailable on any currently-shipping product, but it is going after the same market as a number of more established platforms. MeeGo is interesting: it is a combined effort by two strong industry players which are trying, in the usual slow manner, to build a truly community-oriented development process. For the time being, though, important development decisions are still being made centrally. Recently, a significant decision has come to light: MeeGo will be based on the [Btrfs](<http://lwn.net/Articles/358940/>) file system by default. 

Btrfs is seen as the long-term future of Linux filesystems, representing a much-needed clean break from the legacy filesystem designs we have been using for all these years. With the demise of reiser4 and the unavailability of ZFS, Btrfs would seem to be the only contender for that title. But talk about Btrfs is always framed in "it's not stable yet" terms, with few people willing to commit themselves to an actual date when the filesystem might be ready for production use. It is generally assumed that most cautious users will spend some years running on ext4 before making the jump to Btrfs. The 2.6.34 kernel will be released with this text still guarding the Btrfs configuration entry: 

Btrfs is highly experimental, and THE DISK FORMAT IS NOT YET FINALIZED. You should say N here unless you are interested in testing Btrfs with non-critical data. 

The MeeGo 1.0 release could happen as early as this month; given that, the above words might just seem a bit scary. In fact, they are more scary than they need to be: further on-disk format changes are not expected. The warning, it seems, will be scaled down for 2.6.35. 

So why pick Btrfs for MeeGo? Arjan van de Ven [described the decision](<https://lwn.net/Articles/387199/>) this way: 

It's the future of Linux filesystems. We had a case where the old guard (ext3) is getting retired, and there are two new filesystems on the table (btrfs and ext4). We felt that if we picked ext4, we'd have all the pain of a new filesystem, and we'd then change again a year later to btrfs. 

He went on to describe a number of reasons why Btrfs makes sense for the MeeGo platform, starting with its data integrity features. The copy-on-write design which is at the core of Btrfs has a number of nice attributes, one of which is that users should never, ever see garbage data in files, even in a "pulled out the battery at the worst moment" situation. Device manufacturers, understandably, like that idea. 

The on-disk compression feature is interesting for the MeeGo environment as well. It makes the initial system load take less space, making more available for the users of the device. But, as Arjan points out, manufacturers like it too: a smaller system image takes less time to shovel onto the storage device. 

It would appear that there are a number of plans for the use of the Btrfs snapshot feature, starting with reversible package updates. With snapshots, a device can support a multi-user mode where each user appears to have the entire system to him- or herself. And the "reset to factory defaults" operation becomes a simple operation which does not require a separate recovery partition on the disk. Snapshots are not just for enterprise users anymore. 

There are a number of other advantages, including small-file performance, built-in defragmentation (which is most useful for keeping boot time short), the storage management features, and more. In short, there's no doubt that Btrfs offers a useful set of features for any distribution; it's not hard to see why MeeGo wanted to use it. But that does leave an interesting open question: is Btrfs ready for inclusion into MeeGo, where it will, presumably, be installed onto systems intended for users who aren't looking to become development-stage filesystem testers? 

Btrfs was initially merged for the 2.6.29 kernel; since then, patch activity looks like this: 

> ![\[Btrfs patch rate plot\]](https://static.lwn.net/images/ns/kernel/btrfs-patches.png)

So there is a steady rate of change to the filesystem, significant but not overwhelming. There is a wide range of contributors to this code, though the bulk of the work (by far) has been done by developers from Oracle and Red Hat. There are certainly people using Btrfs in normal use, and Fedora offers it as an experimental option. The mailing list shows a number of oops reports still, and it would appear that the famous ENOSPC issue (where the filesystem reacts poorly when the storage device overflows) is still not entirely solved. Significant feature patches - direct I/O support and RAID 4/5 support, for example - remain unmerged. In summary: Btrfs does not quite have that "it's done" look to it yet. 

That said, it may well be getting close to ready for the sort of restricted and well-tested environment likely to be found in MeeGo deployments. Btrfs will also have stabilized further by the time devices actually start shipping with MeeGo - helped, no doubt, by the work of the MeeGo developers themselves. So, while this decision may appear to be ambitious now, it is not necessarily unreasonable. A dark-horse platform can only be helped by taking advantage of the best technology available to it.
