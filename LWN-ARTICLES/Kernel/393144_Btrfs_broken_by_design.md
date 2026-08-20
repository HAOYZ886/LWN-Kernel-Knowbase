---
title: "Btrfs: broken by design?"
url: https://lwn.net/Articles/393144/
date: "June 22, 2010"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet June 22, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 22, 2010

The Btrfs filesystem is seen by many as the primary Linux filesystem for the next decade or so. It brings a next-generation design and a wide range of features (snapshots, data checksums, internal RAID, etc.) that users have been waiting for. Despite being merged for 2.6.29, Btrfs remains an experimental development, but some of the more adventurous distributions are beginning to offer Btrfs installation options and [Meego has chosen Btrfs](<http://lwn.net/Articles/387196/>) as its default filesystem. So when a filesystem developer started calling Btrfs "broken by design," people took notice. 

Edward Shishkin, perhaps better known for his efforts to keep reiser4 development alive, first [posted some concerns](<https://lwn.net/Articles/393146/>) on June 3\. It seems he ran a simple test: create a new Btrfs filesystem, then create 2048-byte files until space runs out. Others have talked about suboptimal space efficiency in Btrfs before, but Edward was still surprised that he was only able to use 17% of the nominal space in the filesystem before it was reported as being full. Such poor efficiency was, [according to Edward](<https://lwn.net/Articles/393148/>), evidence the Btrfs was "broken by design" and should not be used: 

The first obvious point here is that we *can not* put such file system to production. Just because it doesn't provide any guarantees for our users regarding disk space utilization.... As to current Btrfs code: *NOT ACK*!!! I don't think we need such "file systems". 

Part of the problem comes down to the use of "inline extents" in Btrfs. The core data structure on a Btrfs filesystem is a [B-tree](<https://secure.wikimedia.org/wikipedia/en/wiki/B-tree>) which provides access to all of the objects stored in the filesystem. For larger files, the actual file data is stored in extents, which are pointed to from within the tree. Small extents, though, can be stored in the tree itself, hopefully yielding both better space efficiency and better performance. If these extents are sized inconveniently, though, they can cause a lot of wasted space. There's only room for one 2048-byte inline extent in a B-tree node, leaving 1800 bytes or so of unused space. That is a lot of internal fragmentation - a lot of wasted space. 

As noted in [Chris Mason's response](<https://lwn.net/Articles/393150/>), there are two approaches which can be taken to mitigate this kind of problem. One is to turn off inline extents altogether; Btrfs has a `max_inline=` mount option which can be used for just that purpose. The other approach would be to allow inline extents to be split between tree nodes so that the pieces could be sized to fill those nodes exactly. Btrfs cannot do that, and probably will not be able to anytime soon: 

I didn't put in the splitting simply because the complexity was high while the benefits were low (in comparison with just turning off the inline extents). 

Chris also noted that most of the other variable-size items stored in B-tree nodes - extended attributes, for example - can be split between nodes if need be. So these items should not cause fragmentation problems; it's mainly the inline extents which are at fault there. 

But, as Edward [pointed out](<https://lwn.net/Articles/393151/>), there's more to the problem than inline extents. In his investigations, he's found numerous places where groups of nearly-empty nodes exist; some were less than 1% utilized. That, in all likelihood, is the real source of the worst space utilization problems. To Edward, this behavior is another sign that the algorithms used in Btrfs are all wrong and in need of a redesign. 

Chris [sees it a little differently](<https://lwn.net/Articles/393154/>), though: 

The current code is clearly choosing not to merge two leaves that it should have merged, which is just a plain old bug. 

He has promised to track it down and post a fix. Between the bug fix and turning off inline extents (or, at least, reducing their maximum size), it is hoped that the worst space utilization problems in Btrfs will be no more. 

That fix has not been posted as of this writing, so its effectiveness cannot yet be judged. But, chances are, this is not a case of a filesystem needing a fundamental redesign. Instead, all it needs is more extensive testing, some performance tuning, and, inevitably, some bug fixes. The good news is that the process seems to be working as it should be: these problems have been found before any sort of wide-scale deployment of this very new filesystem.
