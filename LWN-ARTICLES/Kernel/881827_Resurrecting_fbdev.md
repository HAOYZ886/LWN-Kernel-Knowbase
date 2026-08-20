---
title: Resurrecting fbdev
url: https://lwn.net/Articles/881827/
date: "January 19, 2022"
category: "Development model-Maintainers; Device drivers-Graphics"
author: "By Jake Edge January 19, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
January 19, 2022

The Linux framebuffer device (fbdev) subsystem has long languished in something of a purgatory; it was listed as "orphaned" in the `MAINTAINERS` file and saw fairly minimal maintenance, mostly driven by developers working elsewhere in the kernel graphics stack. That all changed, in an eye-opening way, on January 17, when Linus Torvalds merged a [change](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3c750c7b614322a4ab50be0d57a76addb598d51d>) to make Helge Deller the new maintainer of the subsystem. But it turns out that the problems in fbdev run deep, at least according to much of the rest of the kernel graphics community. By seeming to take on the maintainer role in order to revert the removal of some buggy features from fbdev, Deller has created something of a controversy. 

Part of the concern within the graphics community is the accelerated timeline that these events played out on. Deller [posted](<https://lwn.net/ml/linux-kernel/YeG8ydoJNWWkGrTb@ls3530/>) his intention to take over maintenance of the framebuffer on Friday, January 14, which [received an ack](<https://lwn.net/ml/linux-kernel/CAMuHMdWNzDb--Ud1DzGyoqWsgFU9m8YZ6fa1TqfBWtg4pkd==A@mail.gmail.com/>) from Geert Uytterhoeven later that day. Two days later, before any other responses had come in, Deller sent a [pull request](<https://lwn.net/ml/linux-kernel/YeRyfaesC2kxkgZC@ls3530/>) to Torvalds to add Deller as the fbdev maintainer, which was promptly picked up. On January 19, Deller [posted](<https://lwn.net/ml/linux-kernel/20220119110839.33187-1-deller@gmx.de/>) reversions of two patch sets that removed scrolling acceleration from fbdev. In the meantime, those reversions had already been made in Deller's brand new [fbdev Git tree](<https://git.kernel.org/pub/scm/linux/kernel/git/deller/linux-fbdev.git>). 

The patch sets that were being targeted for reversion had been posted and merged some time ago. Daniel Vetter [disabled accelerated scrolling for the framebuffer console](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=39aead8373b3c20bb5965c024dfb51a94e526151>) (fbcon) back at the end of 2020\. At the time, he added a "todo" item to garbage collect the code that supported that accelerated scrolling. Claudio Suarez [posted a patch](<https://lwn.net/ml/linux-kernel/YVXTYqszZix9TxjJ%40gineta.localdomain/>) completing that todo item in September 2021, which was [committed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b3ec8cdf457e5e63d396fe1346cc788cf7c1b578>) in October. On January 13, shortly before deciding to take on maintenance of fbdev, Deller [asked for a reversion](<https://lwn.net/ml/dri-devel/feea8303-2b83-fc36-972c-4fc8ad723bde@gmx.de/>) of the latter patch (or parts of it). 

Once Monday January 17 rolled around, Vetter and others noticed the flurry of activity that had occurred over the weekend and weighed in. Vetter [suggested](<https://lwn.net/ml/linux-kernel/CAKMK7uGdJckdM+fg+576iJXsqzCOUg20etPBMwRLB9U7GcG01Q@mail.gmail.com/>) that it might have been premature to make a maintainer change ""without even bothering to get any input from the people who've been maintaining it before"". In particular, he was concerned about moving fbdev and fbcon to a tree separate from the DRM tree; the subsystem may have been marked as orphaned but the situation is more complicated than that: 

> Because the status isn't entirely correct, fbdev core code and fbcon and all that has been maintained, but in bugfixes only mode. And there's very solid&important reasons to keep merging these patches through a drm tree, because that's where all the driver development happens, and hence also all the testing (e.g. the drm test suite has some fbdev tests - the only automated ones that exist to my knowledge - and we run them in CI [continuous integration]). So moving that into an obscure new tree which isn't even in linux-next yet is no good at all. 
> 
> Now fbdev driver bugfixes is indeed practically orphaned and I very much welcome anyone stepping up for that, but the simplest approach there would be to just get drm-misc commit rights and push the oddball bugfix in there directly. 

Beyond that, Jani Nikula was [taken aback](<https://lwn.net/ml/linux-kernel/87o84a63hy.fsf@intel.com/>) by the whirlwind pace of the changes. In particular, he was not happy to see the reversions being made in the new fbdev tree almost immediately, even though the objection was only made a few days earlier. ""I'm heavily in favor of maintainers who are open, transparent, collaborative, who seek consensus through discussion, and only put their foot down when required."" Deller [said](<https://lwn.net/ml/linux-kernel/60fad896-235c-7602-39d1-14691282ebf6@gmx.de/>) that he had just started going through the backlog of patches; ""nothing has been pushed yet"". He said that Nikula should simply ignore the state of the fbdev tree at this point. 

In response to Vetter, Deller [said](<https://lwn.net/ml/linux-kernel/c80ed72c-2eb4-16dd-a7ad-57e9dde59ba1@gmx.de/>) that having a separate tree was not important. He listed four goals for maintaining fbdev going forward: 

>   1. to get fixes which were posted to fbdev mailing list applied if they are useful & correct, 
>   2. to include new drivers (for old hardware) if they arrive (probably happens rarely but there can be). I know of at least one driver which won't be able to support DRM.... Of course, if the hardware is capable to support DRM, it should be written for DRM and not applied for fbdev. 
>   3. reintroduce the state where fbcon is fast on fbdev. This is important for non-DRM machines, either when run on native hardware or in an emulator. 
>   4. not break DRM development 

Vetter [pointed](<https://lwn.net/ml/linux-kernel/CAKMK7uHVHn9apB6YYbLSwu+adEB2Fqp4FM0z582zf4F-v3_GnQ@mail.gmail.com/>) Deller to the [documentation](<https://drm.pages.freedesktop.org/maintainer-tools/getting-started.html>) for coming up to speed on DRM development and for getting [commit rights in the drm-misc tree](<https://drm.pages.freedesktop.org/maintainer-tools/commit-access.html#drm-misc>), which is the proper path for fbdev fixes, he said. After that: 

> I think once we've set that up and got it going we can look at the bigger items. Some of them are fairly low-hanging fruit, but the past 5+ years absolutely no one bothered to step up and sort them out. Other problem areas in fbdev are extremely hard to fix properly, without only doing minimal security-fixes only support, so fair warning there. I think a good starting point would be to read the patches and discussions for some of the things you've reverted in your tree. 
> 
> Anyway I hope this gets you started, and hopefully after a minor detour: Welcome to dri-devel, we're happy to take any help we can get, there's lots to do! 

Deller eventually [decided to keep the fbdev tree](<https://lwn.net/ml/linux-kernel/4009cf14-2bb0-a63a-1936-4ac4d757777a@gmx.de/>), though he does [plan to coordinate](<https://lwn.net/ml/linux-kernel/bf2a45c4-5f48-24e6-213f-562e59505f3d@gmx.de/>) with the rest of the graphics development community: 

> I'm not planning to push code to fbdev/fbcon without having discussed everything on dri-devel. Everything which somehow would affect DRM needs to be discussed on dri-devel and then - after agreement - either pushed via the fbdev git tree or the drm-misc tree. 

It is clear there are differences of opinion on how to proceed. The hardware-accelerated scrolling that was removed was dependent on the 2D [bit-blit](<https://en.wikipedia.org/wiki/Bit_blit>) acceleration features of older hardware. But the code that used it in the fbdev drivers was apparently rather buggy; over the years, syzbot repeatedly found problems in that code, which is why it was eventually removed. The DRM subsystem [does not have support for 2D acceleration](<https://blog.ffwll.ch/2018/08/no-2d-in-drm.html>), and will not, due to some serious technical difficulties in doing so. 

On the other hand, Deller and others have graphics hardware that uses the fbdev drivers and, formerly, had reasonable performance using the hardware-accelerated scrolling. That scrolling performance went away when the code was removed, and they would like to get it back. But reverting the removals simply brings back the buggy code. From the perspective of the DRM developers, the right way forward is to create DRM-based drivers for these devices, but Deller and others disagree. 

The larger issue is how the transition has been handled, Vetter [said](<https://lwn.net/ml/linux-kernel/CAKMK7uEb53iu_HxYSnFZ59j=vXQdMvTWT7xosEo85XkAwzDMnA@mail.gmail.com/>) in the reversion thread: 

> The other side is that being a maintainer is about collaboration, and this entire fbdev maintainership takeover has been a demonstration of anything but that. [...] This entire affair of rushing in a maintainer change over the w/e [weekend] and then being greeted by a lot of wtf mails next Monday does leave a rather sour aftertaste. Plus that thread shows a lot of misunderstandings of what's all been going on and what drm can and cannot do by Helge, which doesn't improve the entire "we need fbdev back" argument. 

Vetter strongly believes that if the removed features are to return, the fbdev code needs to be modernized to a point ""where we can still tell distros that enabling it is an ok thing to do and not just a CVE subscription"". In addition, he believes there is a more straightforward path toward improving the scrolling behavior without bringing back all of the problems that syzbot has found: 

> Also wrt the issue at hand of "fbcon scrolling": The way to actually do that with some speed is to render into a fully cached shadow buffer and upload changed areas with a timer. Not with hw accelerated scrolling, at least not if we just don't have full scale development teams for each driver because creating 2d accel that doesn't suck is really hard. drm fbdev compat helpers give you that shadow buffer for free (well you got to set some options). 

But Deller [sees things differently](<https://lwn.net/ml/linux-kernel/408c5dbd-a915-e3fc-fafb-738f6badc91a@gmx.de/>); there are existing drivers that need the support that was removed. He intends to try to restore that support, while also presumably fixing whatever problems syzbot or others find: 

> But in addition fbdev/fbcon is the kernel framework for nearly all existing graphic cards which are not (yet) supported by DRM. They need fbdev/fbcon to show their text console and maybe a simple X server. If you break fbdev for those cards, they are completely stuck. Hopefully those drivers will be ported to DRM, but that's currently not easily possible (or they would be so slow that they are [unusable]). 

The DRM developers seem skeptical that the problems already identified can be addressed, but it would seem that they should be giving Deller some time to do so. The "orphaned" status of fbdev was perhaps not the right choice, though it is clear that the DRM community believed any change to that status would come by way of discussion and agreement, rather than via a surprise weekend takeover. Be that as it may, a new maintainer for a long-unloved part of the kernel should be seen as a good thing. We will have to wait to see how it all works out.
