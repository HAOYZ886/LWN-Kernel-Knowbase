---
title: News from the staging tree
url: https://lwn.net/Articles/351480/
date: "September 9, 2009"
category: "Development model-Driver merging; Staging tree"
author: "By Jake Edge September 9, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
September 9, 2009

The staging tree has made a lot of progress since it [appeared](<https://lwn.net/Articles/285594/>) in June 2008. To start with, the tree itself quickly [moved into the mainline](<https://lwn.net/Articles/301192/>) in October 2008; it also has accumulated more than 40 drivers of various sorts. Staging is an outgrowth of the [Linux Driver Project](<http://www.linuxdriverproject.org>) that is meant to collect drivers, and other "standalone" code such as filesystems, that are not yet ready for the mainline. But, it was never meant to be a ""dumping ground for dead code"", as staging maintainer Greg Kroah-Hartman [put it](<https://lwn.net/Articles/350590/>) in a recent status update. Code that is not being improved, so that it can move into the mainline, will be removed from the tree. 

Some of the code that is, at least currently, slated for removal includes some fairly high-profile drivers, including one from Microsoft that was released with great [fanfare](<http://www.microsoft.com/presspass/features/2009/jul09/07-20linuxqa.mspx>) in July. After a massive cleanup that resulted in more than 200 patches to get the code ""into a semi-sane kernel coding style"", Kroah-Hartman said that it may have to be removed in six months or so: 

Unfortunately the Microsoft developers seem to have disappeared, and no one is answering my emails. If they do not show back up to claim this driver soon, it will be removed in the 2.6.33 release. So sad... 

Microsoft is certainly not alone in Kroah-Hartman's report--which details the status of the tree for the upcoming 2.6.32 merge window--as several other large companies' drivers are in roughly the same boat. Drivers for Android hardware (`staging/android`), Intel's Management Engine Interface (MEI) hardware (`staging/heci`), among others were called out in the report. Both are slated for removal, `android` for 2.6.32, and `heci` in 2.6.33 (presumably). The latter provides an excellent example of how not to do Linux driver development: 

A wonderful example of a company throwing code over the wall, watching it get rejected, and then running away as fast as possible, all the while yelling over their shoulder, "it's required on all new systems, you will love it!" We don't, it sucks, either fix it up, or I am removing it. 

Kroah-Hartman's lengthy report covers more than just drivers that may be removed; it also looks at those that have made progress, including some that should be moving to the mainline, as well as new drivers that are being added to staging. But the list of drivers that aren't being actively worked on is roughly as long as the other two lists combined, which is clearly suboptimal. 

Presumably to see if folks read all the way through, Kroah-Hartman sprinkles a few laughs in an otherwise dry summary. For the `me4000` and `meilhaus` drivers, he notes that there is no reason to continue those drivers ""except to watch the RT guys squirm as they try to figure out the byzantine locking and build logic here (which certainly does count for something, cheap entertainment is always good.)"" 

He also notes several drivers that are in the inactive category, but are quite close to being merge-worthy. He suggests that developers looking for a way to contribute consider drivers such as `asus_oled` (Asus OLED display), `frontier` (Frontier digital audio workstation controller), `line6` (PODxt Pro audio effects modeler), `mimio` (Mimio Xi interactive whiteboard), and `panel` (parallel port LCD/keypad). Each of those should be relatively easy to get into shape for inclusion in the mainline. 

There are a fair number of new drivers being added for 2.6.32, including the Microsoft Hyper-V drivers (`staging/hv`) mentioned earlier, as well as VME bus drivers (`staging/vme`), the industrial I/O subsystem (`staging/iio`), and several wireless drivers (VIA vt6655 and vt6656, Realtek rtl8192e, and Ralink 3090). Also, ""another COW driver"" is being added: the Cowloop copy-on-write pseudo block driver (`staging/cowloop`). 

Two of Evgeniy Polyakov's projects--mistakenly listed in the "new driver" section though they were added in 2.6.30--were also mentioned. The distributed storage (DST) network block device (`staging/dst`), which Kroah-Hartman notes may be ""dead"" is a candidate for removal, while the distributed filesystem POHMELFS (`staging/pohmelfs`) is mostly being worked on out-of-tree. Polyakov [agrees that DST is not needed in the mainline](<http://www.ioremap.net/node/350>), but is wondering about moving POHMELFS out of staging and into `fs/`. Since there are extensive changes on the way for POHMELFS, it is unlikely to move out of staging for another few kernel releases at least. 

There was also praise for the work on various drivers which have been actively worked on over the last few months. Bartlomiej Zolnierkiewicz was singled out for his work on `rt*` and `rtl*` wireless drivers (which put him atop the [list of most active 2.6.31 developers](<http://lwn.net/Articles/348445/>)), along with Alan Cox for work on the `et131x` driver for the Agere gigabit Ethernet adapter. Johannes Berg [noted](<https://lwn.net/Articles/351566/>) that much of Zolnierkiewicz's work on the `rt*` drivers ""will have been in vain"" because of the progress being made by the rt2x00 project. But that [doesn't faze](<https://lwn.net/Articles/351567/>) Zolnierkiewicz: 

The end goal of this work has always been having native rt2x00 support for all those chipsets (as have been explained multiple times). If this means that one day we will delete all Ralink drivers in staging in favor of proper wireless drivers -- fine with me. 

In the meantime (before clean and proper support becomes useful) Linux users are provided with the possibility to use their hardware before it becomes obsolete. 

At least one developer stepped up to work on one of the inactive drivers (`asus_oled`) in the thread. In addition, Willy Tarreau [mentioned](<https://lwn.net/Articles/351568/>) that he had heard from another who was working on `panel`, telling Kroah-Hartman: ""This proves that the principle of the staging tree seems to work"". 

Overall, the staging tree seems to be doing exactly what Kroah-Hartman and others envisioned. Adding staging into the mainline, which raised the profile and availability of those drivers, has led to a fair amount of cleanup work, some of which has resulted in the drivers themselves moving out of staging and into the mainline. Some drivers seem to be falling by the wayside, but one would guess that Kroah-Hartman would welcome them back into the tree should anyone show up to work on them. In the meantime, the code certainly hasn't suffered from whatever fixes various kernel hackers found time to do. Those changes will be waiting for anyone who wants to pick that code back up, even if it is no longer part of staging.
