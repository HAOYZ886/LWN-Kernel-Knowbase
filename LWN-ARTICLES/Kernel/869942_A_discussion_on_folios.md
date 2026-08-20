---
title: A discussion on folios
url: https://lwn.net/Articles/869942/
date: "September 22, 2021"
category: "Memory management-Folios"
author: "By Jake Edge September 22, 2021 LPC"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
September 22, 2021

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2021-Linux_Plumbers_Conference>)

A few weeks ago, Matthew Wilcox might have guessed that his [session](<https://linuxplumbersconf.org/event/11/contributions/976/>) at the [2021 Linux Plumbers Conference](<https://linuxplumbersconf.org/event/11/>) would be focused rather differently. But, as we [reported earlier in September](<https://lwn.net/Articles/868598/>), his [folio](<https://lwn.net/Articles/849538/>) patch set ran into some, perhaps unexpected, opposition and, ultimately, did not land in the mainline for 5.15. Instead of discussing how to use folios as part of the [File Systems microconference](<https://linuxplumbersconf.org/event/11/sessions/111/>), he led a discussion that was, at least in part, on the path forward for them. 

Wilcox began by noting that the folio patches had not been merged and that he did not have clear direction from Linus Torvalds about what "needs to be changed in order to make it acceptable to him". That is a rather different outcome than Wilcox had been hoping for, so the session was not going to be about "what you need to do in order to enable your filesystems to go faster" using folios. "That's not where we are." 

[ ![\[Matthew Wilcox\]](https://static.lwn.net/images/2021/folio-wilcox-sm.png) ](<https://lwn.net/Articles/870012/>)

Instead, though, it would be useful to talk about what filesystems need to do in order to use larger pages in the page cache. That will make filesystem I/O more efficient, he said. The best thing that developers can do is to convert the filesystems that still use buffer heads to use iomap for I/O, at least for block-based filesystems. Network filesystems should use David Howells's [netfs_lib](<https://www.kernel.org/doc/html/latest/filesystems/netfs_library.html>). In both cases, those APIs will isolate the filesystems from most of the work that Wilcox is doing, he said with a chuckle. 

Both of those APIs insulate filesystems from the page cache; for example, they use bytes for sizes, rather than pages. Any filesystems that are working with the page cache directly should look at using iomap or netfs_lib instead. There are artificial filesystems in the kernel, such as procfs, that do not deal with the page cache at all; they fall outside of the discussion, since the focus is on improving the page cache. 

There are features that the buffer-head interface has, which iomap currently lacks. He has been talking with filesystem developers about what needs to be added so that block-based filesystems can be converted to use it. There are features that iomap needs to in order to support both [fscrypt](<https://www.kernel.org/doc/html/latest/filesystems/fscrypt.html>) and [fs-verity](<https://www.kernel.org/doc/html/latest/filesystems/fsverity.html>). There are also some things that buffer heads can do in the I/O completion path, which are lacking in iomap, but there are "no fundamental limitations", he said. As always, though, there are limitations in terms of developer time 

Howells briefly described the work he is doing on making fscrypt work with network filesystems and, in particular, to use folios in that work. The idea is to put fscrypt and the page-cache handling code into a separate library for network filesystems to use. Currently, AFS and Ceph are using it. When a network filesystem receives encrypted data from the server, it should not be storing it in the page cache unencrypted, but the code to keep the data encrypted should not be replicated in the six different network filesystems. His work hit something of a wall when Torvalds did not merge the folio patches, however. 

#### Help wanted?

[ ![\[Discussion\]](https://static.lwn.net/images/2021/folio-discuss-sm.png) ](<https://lwn.net/Articles/870013/>)

Josef Bacik asked how the filesystem developers could help Wilcox; he knows that moving to iomap is part of that and that is ongoing work for Btrfs. But he wondered if there were concrete steps the filesystem developers could take to support Wilcox's efforts; "sometimes it is patch review, other times it is design review, [...] what do you wish that we would do to help you with all of this?" 

Wilcox said that he did not think there was a lot to do on the filesystem side. He has "imposed on the XFS developers extensively" and they helped work out some of his misconceptions. They also helped ferret out problems in his iomap changes; iomap was originally cut from the bottom of XFS and turned into common code, he noted. If there were going to be an effort to make buffer-head-using filesystems handle larger pages, he would need similar help from the developers of one of those filesystems, he said. But the consensus seems to be that the right path is to minimize the use of buffer heads, so that probably won't be needed. On the flipside, though, Wilcox feels that he and the XFS developers owe the developers of other filesystems some assistance in converting to iomap. But he has already seen some of that going on in the current efforts to convert direct I/O for Btrfs to use iomap; it is all proceeding collaboratively. 

Bacik said that he wasn't necessarily asking as a Btrfs developer, rather as a kernel developer. These kinds of changes often come down to a single person pushing them forward even though lots of others want the changes in the kernel; that one person can sometimes have a rough time of it, he said. 

Ted Ts'o asked whether help with benchmarking would be useful. Wilcox said that he does not do much in the way of benchmark testing; he runs xfstests many times a day, but is "not set up to do performance testing". Both Ts'o and Bacik said they were set up to do that kind of testing, so if Wilcox could point them at a Git tree, testing with and without folios could be done regularly. 

In something of a tangent, Ts'o noted that fscrypt and fs-verity were written as libraries of a sort, so that they could be adopted more easily by other filesystems. But the combination of the two was not really made as accessible. He has considered providing a sort of generic pipeline for operations (e.g. decryption, verification, decompression) on filesystem data coming from block devices or over the network that could be combined in arbitrary ways. 

He suggested to Howells that the next version of the work he is doing on fscrypt support for network filesystems provide some kind of ability to stack operations of this sort. Howells described what he has been doing to make fscrypt work with some of the network filesystems using larger pages. That kind of work will be needed for ext4, Ts'o said, so that it can move to using iomap as well. They agreed to pick up that discussion elsewhere when more of the appropriate people were present. 

#### Folio future

Wilcox said that over the previous few days he and Kent Overstreet had been talking about getting folios merged. Overstreet said that he thought progress had been made in the discussion on the linux-kernel mailing list over the last week or so. He said that the objections to folios turned out to not really apply to the actual code that was proposed for 5.15 in many cases. In particular, the patches did not really touch anonymous pages, which was an area of contention, he said. 

Overstreet suggested that Wilcox consult with Andrew Morton to see where things stand because Overstreet felt that the objections were largely on where folios were headed rather than what was actually posted. Beyond that, later in the day, Overstreet [posted](<https://lwn.net/ml/linux-kernel/YUpNLtlbNwdjTko0@moria.home.lan/>) a request to Torvalds to reconsider the original folios patch set for inclusion. But it turns out that the situation is more complicated than Overstreet perceived. In particular, Johannes Weiner, who disagrees strongly with the direction of folios, at least on the memory-management side, had [objections](<https://lwn.net/ml/linux-kernel/YUtHCle%2FgiwHvLN1@cmpxchg.org/>) to the existing patch set and not just the direction they are headed in. 

Ts'o said that someone had remarked to him that the memory-management developers may not have actually looked closely at folios until the pull request came in. That may be due to a "cultural expectations that stuff in MM takes forever and a day". He wondered if there was a way to work with those developers to determine what the review procedure is for patches of this nature. There may be a process question as to how the subsystem as a whole makes decisions. That would help everyone understand the ground rules on how to get consensus; he does not believe there is a consensus within the memory-management community on folios, at least yet. 

There is no one who is really working on fixing "the `struct page` mess" at this point, Overstreet said, which is perhaps part of the problem; Wilcox's work is the closest the community has come to that. But the reason that this discussion has been so contentious is that the state from different subsystems is all jumbled together in `struct page`, Overstreet said. There is no real overarching design for unwinding that. 

He said that everyone sees the mess, but that each person sees a different aspect of that mess; folios are cleaning up one part, but some are unhappy that their part of it is not being cleaned up right away. An overall plan that could show the end goal and where each part gets cleaned up would go a long way toward resolving these problems, he said. That ended the session, but clearly the overall discussion will continue. 

Interested readers can watch the [video of the session](<https://youtu.be/U6HYrd85hQ8?t=1475>) over at YouTube.
