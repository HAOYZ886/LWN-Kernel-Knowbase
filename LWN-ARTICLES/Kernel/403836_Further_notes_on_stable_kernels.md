---
title: Further notes on stable kernels
url: https://lwn.net/Articles/403836/
date: "September 8, 2010"
category: "Development model-Stable tree; Releases-Stable updates"
author: "By Jonathan Corbet September 8, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 8, 2010

[Last week's article on stable kernels](<http://lwn.net/Articles/402512/>) drew a number of comments, both public and private. Those comments suggested a couple of other ways of looking at how the stable tree works and how patches get into it. That, in turn, has inspired this follow up look at the stable kernel process. 

A certain amount of unhappiness was expressed regarding the tables of the most active stable contributors. Those tables attribute stable contributions to the individuals who write the patches. Things were done this way for two reasons: (1) the patch author is, indeed, the person who fixed the bug, and (2) that is the information which is available in the stable kernel repository. It made sense, your editor thought, to assign credit - for the fix, but, also, potentially, for the bug which required the fix - in this way. 

It turns out that a number of people see stable contributions in a different way. The real credit, they say, belongs to the person who notices that a patch fixes a bug in stable kernels and ensures that the fix gets directed to the stable kernel maintainer. There are people, often those working on maintaining distributor kernels, who spend a lot of time watching the patch stream and looking for just this kind of fix. It is a lot of work, and the people who do that work certainly deserve credit for the service they are performing for the community. 

Your editor would be delighted to be able to produce a table crediting this type of stable contributor. Unfortunately, the electronic trail needed to create this table simply does not exist. One could try to play games by looking at how the patch tags differ between the mainline and stable versions of the fix; there will often be an extra signoff or `Cc:` tag naming the person who forwarded the patch to the stable tree. But such schemes will be approximate and error-prone. If we really want to track and credit developers who flag patches for the stable tree, we almost certainly need to add a new patch tag making that credit explicit in every patch. 

A related complaint came, via private mail, from a subsystem maintainer; his point of view was that the subsystem maintainers are the people doing the real legwork to get important fixes into the stable tree. A diligent maintainer will be evaluating all patches as they are merged into the subsystem tree, catching those which have stable kernel implications and directing them accordingly. He suggested a study to evaluate the percentage of stable patches coming out of each subsystem tree as a way to identify which maintainers are on top of things. 

Your editor, intrigued by that idea, ran a quick study. The table below shows numbers for some selected subsystems for the 2.6.32 stable series. Since 2.6.32 is still under maintenance, it will have received patches from all of the mainline releases from 2.6.33 to the present. For each subsystem, we can look at how many patches have gone into the mainline (through 2.6.36-rc3) and how many of those went into the stable series. The results look like this: 

> Subsystem | Patches | Pct  
> ---|---|---  
> (mainline)| (stable)  
> `fs/ext4` | 216 | 90 | 42%  
> `fs/btrfs` | 155 | 42 | 27%  
> `drivers/usb` | 1003 | 112 | 11%  
> `arch/x86` | 1877 | 176 | 9%  
> `drivers/acpi` | 291 | 24 | 8%  
> `mm` | 602 | 48 | 8%  
> `kernel` | 1471 | 96 | 7%  
> `sound` | 1369 | 88 | 6%  
> `fs/ext3` | 58 | 3 | 5%  
> `drivers/scsi` | 1054 | 51 | 5%  
> `net` | 2324 | 98 | 4%  
> `drivers/input` | 381 | 13 | 3%  
> `arch/powerpc` | 917 | 18 | 2%  
> `drivers/media` | 1705 | 26 | 2%  
> `block` | 182 | 3 | 2%  
> `arch/arm` | 3221 | 19 | <1%  
> `tools` | 873 | 3 | <1%  
  
At the upper end of the table, it is unsurprising to find the ext4 and btrfs filesystems showing a high percentage of stable patches. Both of those filesystems are undergoing heavy stabilization work at the present, so it makes sense that the bulk of the changes merged will be important fixes. The relatively small percentage of ext3 changes going into the stable tree was interesting; a quick check shows that many of the ext3 changes which did not go to stable reflect API changes in the VFS and disk quota code. That said, it also appears that a small number of fixes might have fallen through the cracks. 

It's hard to draw conclusions from much of the rest of the table; different subsystems will naturally vary in the ratio of fixes to new features, so they will never have the same percentage of patches going into the stable tree. That said, there do seem to be some real variations in how many fixes are being directed to stable by the subsystem maintainers. One might, for example, wonder if a few more than 19 of the 3221 changes to the ARM architecture could have qualified for the stable tree. 

This maintainer also pointed out one other aspect of the problem: the maintainer's real job is often to say "no" in the same way as with mainline patches. It seems that some developers have an expansive view of of which changes are suitable for the stable tree, so they flag patches which are too large and invasive, or which do not actually fix serious bugs. In these cases, the maintainer must remove the stable tag and keep the patch from going in that direction. Needless to say, this kind of activity is even harder to track, so there will be no "stable rejections" table. 

In any case, maintainers needing to turn away marginal stable patches seems like the right kind of problem to have. Bugs are annoying in the best of times, but they are doubly annoying when a fix exists but is not distributed to people who need it. The stable tree seems to be doing a good job of getting those fixes out; that makes Linux better for all of us.
