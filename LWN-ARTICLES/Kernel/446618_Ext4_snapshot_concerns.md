---
title: Ext4 snapshot concerns
url: https://lwn.net/Articles/446618/
date: "June 8, 2011"
category: "Filesystems-ext4"
author: "By Jonathan Corbet June 8, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 8, 2011

The next3 filesystem patch, which added snapshots to the ext3 filesystem, appeared just over one year ago; LWN's [discussion of the patch](<https://lwn.net/Articles/387231/>) at the time concluded that it needed to move forward to ext4 before it could possibly be merged. That change has been made, and recent [ext4 snapshot](<https://lwn.net/Articles/446481/>) patches are starting to look close to being ready for merging into the mainline. That has inspired the airing of new concerns which may slow the process somewhat. 

One [complaint](<https://lwn.net/Articles/446620/>) came from Josef Bacik: 

I probably should have brought this up before, but why put all this effort into shoehorning in such a big an invasive feature to ext4 when btrfs does this all already? Why not put your efforts into helping btrfs become stable and ready and then use that, instead of having to come up with a bunch of hacks to get around the myriad of weird feature combinations you can get with ext4? 

Snapshot developer Amir Goldstein's [response](<https://lwn.net/Articles/446622/>) is that his employer (CTERA Networks) wanted the feature in ext4. The feature is shipping in products now, and btrfs is still not seen as stable enough to use in that environment. 

There are general concerns about merging another big feature into a filesystem which is supposed to be stable and ready for production use. Nobody wants to see the addition of serious bugs to ext4 at this time. Beyond that, the snapshot feature does not currently work with all variants of the ext4 on-disk format. There are a number of ext4 features which do not currently play well together, leading Eric Sandeen to [worry](<https://lwn.net/Articles/446625/>) about where the filesystem is going: 

If ext4 matches the lifespan of ext3, in 10 years I fear that it will look more like a collection of various individuals' pet projects, rather than any kind of well-designed, cohesive project. How long can we really keep adding features which are semi- or wholly- incompatible with other features? 

Consider this a cry in the wilderness for less rushed feature introduction, and a more holistic approach to ext4 design... 

Ext4 maintainer Ted Ts'o has [responded](<https://lwn.net/Articles/446626/>) with a rare (for the kernel community) admission that technical concerns are not the sole driver of feature-merging decisions: 

It's something I do worry about; and I do share your concern. At the same time, the reality is that we are a little like the Old Dutch Masters, who had take into account the preference of their patrons (i.e., in our case, those who pay our paychecks :-). 

In this case, he thinks that there are a lot of people who are interested in the snapshot feature. He [worried](<https://lwn.net/Articles/446627/>) that companies like CTERA could move away from ext4 if it can't be made to meet their needs. So his plan is to merge snapshots once (1) the patches are good enough and (2) it looks like there is a plan to address the remaining issues.
