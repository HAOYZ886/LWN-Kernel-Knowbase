---
title: "On-disk format robustness requirements for new filesystems"
url: https://lwn.net/Articles/796687/
date: "August 19, 2019"
category: Filesystems
author: "By Jonathan Corbet August 19, 2019"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 19, 2019

The "Extendable Read-Only File System" (or "EROFS") was first [posted](<https://lore.kernel.org/lkml/1527764767-22190-1-git-send-email-gaoxiang25@huawei.com/>) by Gao Xiang in May 2018; it was merged into the staging tree for the 4.19 release. There has been a steady stream of work on EROFS since then, and its author now thinks that it is ready to move out of staging and join the other official filesystems in the kernel. It would seem, though, that there is one final hurdle that it may have to clear: robustness in the face of a corrupted on-disk filesystem image. That raises an interesting question: to what extent do new filesystems have to exhibit a level of robustness that is not met by the filesystems that are currently in heavy use? 

As suggested by its name (and its acronym), EROFS is a read-only filesystem. It was developed at Huawei, and is intended for use in Android systems. EROFS is meant to differ from existing read-only filesystems in the area of performance; it uses a special compression algorithm that creates fixed-length blocks that, it is claimed, allows random access to compressed data with a minimum of excess I/O and decompression work. Details can be found in [this USENIX paper [PDF]](<https://www.usenix.org/system/files/atc19-gao.pdf>) published in July. 

Gao has made several requests in recent times to move EROFS out of the staging tree; [the latest](<https://lwn.net/ml/linux-kernel/20190817082313.21040-1-hsiangkao@aol.com/>) was posted on August 17\. It read: 

In the past year, EROFS was greatly improved by many people as a staging driver, self-tested, betaed by a large number of our internal users, successfully applied to almost all in-service HUAWEI smartphones as the part of EMUI 9.1 and proven to be stable enough to be moved out of staging. 

([EMUI](<https://en.wikipedia.org/wiki/Huawei_EMUI>) is Huawei's version of Android.) 

It would seem that there is little opposition to this move in general. As part of reviewing the code, though, Richard Weinberger [noticed](<https://lwn.net/ml/linux-kernel/1163995781.68824.1566084358245.JavaMail.zimbra@nod.at/>) that the code generally trusts the data it reads from disks, often failing to check it for reasonableness. He quickly found a way to create a malformed filesystem that would put the kernel into an infinite loop, creating a system that is a bit more read-only than anybody had in mind. The problem was fixed just as quickly, but not before starting a discussion on whether robustness against hostile filesystem images should be a requirement for new filesystems entering the kernel. 

Nobody disagrees that it would be a good thing if a filesystem implementation would do the right thing when faced with a hostile (or merely corrupt) filesystem image; that would make it possible to allow unprivileged users to mount filesystems without fear of handing over the keys to the entire system, for example. But, as Ted Ts'o [pointed out](<https://lwn.net/ml/linux-kernel/20190818151154.GA32157@mit.edu/>), heavily used, in-kernel filesystems like ext4 and XFS don't meet that standard now, so requiring new filesystems to reach that level of robustness is presenting them with a higher bar: 

So holding a file system like EROFS to a higher standard than say, ext4, xfs, or btrfs hardly seems fair. There seems to be a very unfortunate tendency for us to hold new file systems to impossibly high standards, when in fact, adding a file system to Linux should not, in my opinion, be a remarkable event. 

In the case of EROFS, as Chao Yu [pointed out](<https://lwn.net/ml/linux-kernel/8319896b-22c1-0550-f464-f4419a70202e@kernel.org/>), the intended use case makes this kind of robustness less important. The Android system images shipped in this filesystem format will be verified with a system like [dm-verity](<https://source.android.com/security/verifiedboot/dm-verity>), so the filesystem implementation should not be confronted with anything other than signed and verified images. Even so, the EROFS developers agree that this kind of bug should be actively sought out and fixed. 

It seems that views about robustness against bad images vary somewhat among filesystem developers. With regard to these bugs in ext4, Ts'o said that ""while I try to address them, it is by no means considered a high priority work item"". He characterized the approach of the XFS developers as being similar. Christoph Hellwig [disagreed strongly](<https://lwn.net/ml/linux-kernel/20190818155812.GB13230@infradead.org/>) with that claim, though, saying that XFS developers work hard to handle corrupt filesystem images, ""although there are of course no guarantees"". Eric Biggers [asserted](<https://lwn.net/ml/linux-kernel/20190818161638.GE1118@sol.localdomain/>) that dealing with robustness issues should be mandatory, ""but I can understand that we don't do a good job at it, so we shouldn't hold a new filesystem to an unfairly high standard relative to other filesystems"". 

Hellwig arguably [took the strongest position](<https://lwn.net/ml/linux-kernel/20190818162201.GA16269@infradead.org/>) with regard to the standards that should be applied to new filesystems: 

We can't really force anyone to fix up old file systems. But we can very much hold new ones to (slightly) higher standards. That's the only way to get the average quality up. Same as for things like code style - we can't magically fix up all old stuff, but we can and usually do hold new code to higher standards. 

What those higher standards should be was not spelled out. They probably do not extend to absolute robustness against corrupt filesystem images, but it seems that developers would like to see at least an effort made in that direction. As Biggers [put it](<https://lwn.net/ml/linux-kernel/20190818172938.GA14413@sol.localdomain/>): 

If the developers were careful, the code generally looks robust, and they are willing to address such bugs as they are found, realistically that's as good as we can expect to get. 

Whether EROFS meets the "looks robust" standard is a bit controversial at the moment. On the other hand, there is little doubt that the EROFS developers are willing and able to fix bugs quickly as they are reported. For the purposes of moving EROFS into the kernel proper, chances are that will be good enough. Unless some other show-stopping issue comes up, this little snag seems unlikely to keep this code from graduating out of the staging tree. Future filesystem developers will want to take notice, though, that reviewers will be paying more attention to robustness against on-disk image corruption than they have in the past.
