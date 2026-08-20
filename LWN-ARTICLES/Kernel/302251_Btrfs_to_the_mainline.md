---
title: "Btrfs to the mainline?"
url: https://lwn.net/Articles/302251/
date: "October 8, 2008"
category: "Development model; Filesystems-Btrfs"
author: "By Jake Edge October 8, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
October 8, 2008

One of the kernel projects that seems to be attracting a fair amount of attention these days is the new, copy-on-write filesystem, [Btrfs](<http://btrfs.wiki.kernel.org/>). While still rather immature--the disk format is slated to be finalized by the end of the year--Btrfs has reached a point where lead developer Chris Mason wants to [start talking about when to merge it](<https://lwn.net/Articles/302256/>) into the mainline. Some are advocating moving quickly, while others are a bit more skeptical that merging it will lead to faster development. 

Merging Btrfs would have a number of advantages, but more eyes is what Mason is seeking: 

But, the code is very actively developed, and I believe the best way to develop Btrfs from here is to get it into the mainline kernel (with a large warning label about the disk format) and attract more extensive review of both the disk format and underlying code. 

The Btrfs developers are committed to making the FS work and to working well within the kernel community. I think everyone will be happier with the final result if I am able to attract eyeballs as early as possible. 

Typically, kernel code is not merged until it is ready, but an argument can be made that filesystems, [like device drivers](<http://lwn.net/Articles/298570/>), are sufficiently isolated from the rest of the kernel that an early inclusion will do little harm. Also, a kind of precedent was set by the early "merge" of ext4, though that was an evolution of the existing ext3 filesystem, while Btrfs is entirely new. Andrew Morton has been [encouraging](<https://lwn.net/Articles/302255/>) Mason to get Btrfs ""into linux-next asap and merge it into 2.6.29"". He describes his reasoning: 

My thinking here is that btrfs probably has a future, and that an early merge will accelerate its development and will broaden its developer base. If it ends up failing for some reason, well, we can just delete it again. 

For various reasons this approach often isn't appropriate as a general policy thing, but I do think that Linux has needed a new local filesystem for some time, and btrfs might be The One, and hence is worth a bit of special-case treatment. 

Adrian Bunk is [not convinced](<https://lwn.net/Articles/302297/>) that an early merge will bring the benefits that Morton is touting. He points to an [early ext4 development plan](<http://lwn.net/Articles/189950/>), noting that the timelines outlined in that message were, perhaps, overly optimistic. ""When comparing with what happened in reality it kinda disproves your 'acceleration' point"". 

There is a difference, though, between ext4 and Btrfs, that Serge Hallyn [points out](<https://lwn.net/Articles/302299/>): 

OTOH, maybe it's just me, but I think there is more excitement around btrfs. Myself I'm dying for snapshot support, and can't wait to try btrfs on a separate data/scratch partition (where i don't mind losing data). btrfs and nilfs - yay. Ext4? <yawn> That can make all the difference. 

The original timeline showed mid-2007 as a target for a stable ext4 filesystem, but the project overshot that by a year or so. A recent [patch](<http://lwn.net/Articles/302148/>) proposes renaming ext4dev to ext4 because it ""is getting stable enough that it's time to drop the 'dev' prefix"". Unexpected difficulties led to ext4 development taking longer, as Mason [describes](<https://lwn.net/Articles/302303/>): 

Ext4 has always had to deal with the ghost of ext3. Both from a compatibility point of view and everyone's expectations of stability. I believe that most of us underestimated how difficult it would be to move ext4 forward. 

Many seem to think that Btrfs is different, but it still has a ways to go. Currently, it does not handle I/O errors very well, while running out of space on the disk can be fatal. But it is getting close to usable--at least for testing and benchmarking. Getting the code into the mainline would cause more folks to look at it, as well as test various filesystem changes against it. Mason gives an example of how that can work: 

For example, see the streaming write patches I sent to fsdevel last week. I wouldn't test against ext4 as often if I had to hunt down external repos just to get something consistent with the current development kernels. ext4 in mainline makes it much easier for me to kick the tires. 

Btrfs has an [aggressive schedule](<http://btrfs.wiki.kernel.org/index.php/Development_timeline>) that targets a 1.0 release this year. The focus of that release is to nail down the on-disk format so that changes after that point will be backward compatible. Given that 2.6.29 will likely be released in early to mid-2009, it seems quite possible that Btrfs will be "merge-worthy" by then, which means that it really is not premature to start considering it now.
