---
title: Examining exFAT
url: https://lwn.net/Articles/797963/
date: "August 30, 2019"
category: "Filesystems-exFAT"
author: "By Jonathan Corbet August 30, 2019"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 30, 2019

Linux kernel developers like to get support for new features — such as filesystem types — merged quickly. In the case of the [exFAT](<https://en.wikipedia.org/wiki/ExFAT>) filesystem, that didn't happen; exFAT was created by Microsoft in 2006 for use in larger flash-storage cards, but there has never been support in the kernel for this filesystem. Microsoft's recent [announcement](<https://cloudblogs.microsoft.com/opensource/2019/08/28/exfat-linux-kernel/>) that it wanted to get exFAT support into the mainline kernel would appear to have removed the largest obstacle to Linux exFAT support. But, as is so often the case, it seems that some challenges remain. 

For years, the Linux community mostly ignored exFAT; it was a proprietary format overshadowed by an unpleasant patent cloud. A Linux driver existed, though, and was shipped as a proprietary module on various Android devices. In 2013, the code for this driver [escaped into the wild](<https://lwn.net/Articles/560424/>) and was posted to a GitHub repository. But that code was never actually released under a free license and the patent issues remained, so no serious effort to upstream it into the mainline kernel was ever made. 

The situation stayed this way for some years. Even Microsoft's [decision to join the Open Invention Network](<https://lwn.net/Articles/768054/>) (OIN) in 2018 did not change the situation; exFAT, being outside the OIN Linux System Definition, was not covered by any new patent grants. Some people pointed this out at the time, but it didn't raise a lot of concern. Most people, it seemed, had simply forgotten about exFAT, which has a relatively limited deployment overall. 

In July of this year, though, Valdis Klētnieks [posted](<https://lwn.net/ml/linux-kernel/21080.1562632662@turing-police/>) that he had ""beaten into shape"" the exFAT code and wondered how it might be upstreamed. The ensuing discussion made it clear that the patent issues were still a show-stopper for inclusion; that discussion also included a couple of pointed suggestions to the Microsoft employees on the list that perhaps they could help to change that situation. By all appearances, that prod [started an internal discussion](<https://lwn.net/ml/linux-kernel/SN6PR2101MB10721504993B62F1D6EBD693A0F10@SN6PR2101MB1072.namprd21.prod.outlook.com/>) that ended with Microsoft agreeing with the addition of exFAT to the mainline kernel. 

One never really knows what is going on in large companies. The exclusion of exFAT from Microsoft's commitment to OIN looked like a deliberate, old-time Microsoftian act, but it now looks likely that opening up exFAT is just one of those things that nobody thought about until it was brought to their attention. 

Greg Kroah-Hartman wasted no time in taking Klētnieks's code and [proposing it for addition](<https://lwn.net/ml/linux-kernel/20190828160817.6250-1-gregkh@linuxfoundation.org/>) to the staging tree for further work, of which it is said to need a fair amount. That drew an equally quick [objection](<https://lwn.net/ml/linux-kernel/20190829062340.GB3047@infradead.org/>) from Christoph Hellwig, who said it would be better to ""just review the damn thing and get it into the proper tree"". He is unhappy about how filesystems have been handled in the staging tree in the past, and mentioned the [handling of the EROFS filesystem](<https://lwn.net/Articles/796687/>) as a particular sore point. That sparked a whole subthread on the remaining concerns about EROFS that has little to do with exFAT. 

The code quality of the exFAT implementation is of concern generally; that is the kind of thing that can be improved over time in the staging tree. But there are a couple of deeper issues that could yet prove to be a sticking point for exFAT. One is [a complaint](<https://lwn.net/ml/linux-kernel/20190829121435.bsl5cnx7yqgakpgb@pali/>) from Pali Rohár that the posted [specification](<https://docs.microsoft.com/en-us/windows/win32/fileio/exfat-specification>) is incomplete. In particular, he said, the "TexFAT" extension is not documented. As Klētnieks [pointed out](<https://lwn.net/ml/linux-kernel/81682.1567082044@turing-police/>), this extension seems to be used only by Windows CE, so it may prove to be a feature that the rest of the world can do without. 

The bigger concern, perhaps, is that this filesystem module should not exist at all, so tweaking it will not help the situation. As Hellwig [put it](<https://lwn.net/ml/linux-kernel/20190829094136.GA28643@infradead.org/>): 

It basically is a reimplementation of fs/fat/ not up to kernel standards with a few indirections thrown in to also support exfat. So no amount of work on this codebase is really going to bring us forward. 

The right course, he said, is to just add the necessary support to the kernel's existing VFAT filesystem. Kroah-Hartman [replied](<https://lwn.net/ml/linux-kernel/20190829095019.GA13557@kroah.com/>) that he had tried to do that once ""a few years ago"" and concluded that it wouldn't work. But, he said, it may well be easier now that the specification has been posted. If exFAT support were to be reimplemented entirely, perhaps as part of the existing VFAT code, the staging version could simply be deleted once it outlived its usefulness. It would not be the first time such a thing had happened. 

Hellwig is clearly [not convinced](<https://lwn.net/ml/linux-kernel/20190829103749.GA13661@infradead.org/>) that things will play out that way, but his concerns may not be enough to keep the exFAT code out of the staging tree. Whether that step is taken or not, though, there is clearly some work to be done before exFAT truly becomes a part of the mainline kernel. But, then, after thirteen years out in the cold, there is probably no point in being in a hurry to get full support upstream now.
