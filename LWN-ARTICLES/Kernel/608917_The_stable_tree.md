---
title: The stable tree
url: https://lwn.net/Articles/608917/
date: "August 20, 2014"
category: "Development model-Stable tree"
author: "By Jonathan Corbet August 20, 2014 Kernel Summit 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 20, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

While much attention goes into the development of the mainline kernel, most users end up running something based on the stable tree, which incorporates important fixes after the mainline process has moved on. A session during the first day of the 2014 Kernel Summit, led by Greg Kroah-Hartman and Li Zefan, discussed the management of the stable tree and how things could be done better. In the end, it seems, there is little room for improvement at the moment. 

Greg started off by noting that, one year ago, he had asked developers to do a better job of ensuring that important patches make it to the stable tree. In that time, he said, things have gotten a lot better, to the point that he does not know of a single subsystem that is not doing it right. Important fixes from all over the kernel are now finding their way into the stable tree. 

That said, he did note that he found it a bit strange that the 3.17-rc1 release had 200 patches marked for backporting to the stable tree. If the patches are truly important fixes, they should not wait for -rc1; instead, they should go into the mainline quickly. Dave Airlie suggested that the problem is that "people are scared of Linus" and are holding their fixes [![\[Li Zefan\]](https://static.lwn.net/images/conf/2014/ks/Li-Zefan-sm.jpg)](<https://lwn.net/Articles/608920/>) for -rc1 whenever they are in doubt. Greg noted that some of those patches are non-critical things like new device IDs, but others should perhaps be tracked into the mainline more quickly. 

Greg announced that he will no longer be managing the 3.4 long-term stable kernel; that task has been passed over to Li. 

Dmitry Torokhov asked if there weren't perhaps too many patches going into the stable trees. Might it make sense, he asked, to split stable into two trees, one of which would be only fixes while the other would handle new-device enablement. Greg responded that, when looked at individually, every patch going into stable seems to make sense when it is added. As a percentage of the total patch flow into the kernel, even hundreds of patches going into the stable tree are a drop in the bucket. 

Eric Biederman said that, over time, the stable kernels really do get more stable. Chris Mason added that, at Facebook, they have not seen a single regression caused by a patch added to the 3.10 stable tree. 

Rusty Russell asked if developers should be even more aggressive about sending patches to the stable tree. Perhaps fixes for performance regressions should go there as well? Greg responded that performance fixes are already accepted into the stable trees; they seem to be wanted, so he will take them. Mel Gorman added that there has been a determined effort to get important performance fixes into the stable kernels in an attempt to reduce the performance deltas between various kernel versions. In particular, it is not desirable that stable releases perform notably worse than distributor kernels, which tend to have those performance fixes mixed in. 

The subject of the extended stable kernels maintained by developers at Canonical came up. Greg said that, for the most part, those kernels could be ignored; Canonical is doing its own thing there. Peter Anvin complained about the use of the same numbering space for Canonical's kernels; that, he said, can be confusing. He would like to see those kernels named in a way that makes it [![\[Greg KH\]](https://static.lwn.net/images/conf/2014/ks/Greg-Kroah-Hartman-sm.jpg)](<https://lwn.net/Articles/608920/>) clear that they are not "official" stable releases. In response to a question about hosting those kernels on kernel.org, Greg noted that he will hand a stable kernel space over to any community developer in good standing. Canonical's developers, he said, are not that; they do not do work within the community. Since he cannot trust their work, he doesn't want their kernels on kernel.org. 

Andy Lutomirski asked how he could mark a patch as applying only to the 3.16 kernel; the answer is to put "`# v3.16+`" in the patch tags. The use of the "`Fixes:`" tag can also help a lot, Greg said. But, he added, if a patch has a "`Fixes:` tag for an old bug, that patch should be directed toward the stable tree; apparently that does not always happen. Rusty asked about patches for theoretical bugs that have not been reported by actual users; Greg said they can be submitted, and he takes about a quarter of them. 

What about the [Long-Term Support Initiative](<https://lwn.net/Articles/484337/>) kernels? Darren Hart noted that kernel developers tend to scoff at those kernels, but, from his experience, they tend to reduce duplicate backports and are thus useful. Greg said that they can be seen as an "embedded Linux enterprise kernel" and that, most of the time, it's better to just use a newer kernel. Christoph Hellwig complained about the addition of out-of-tree code to those kernels, but Greg said that [LTTng](<https://lwn.net/Articles/491510/>) is the only out-of-tree patch in LTSI, and that everybody in the embedded space needs it. Mark Brown said that the short merge window for LTSI is problematic; Greg responded that patches could be sent to him anytime and they would get into LTSI when the window opened. 

Getting back to the long-term kernels maintained by Greg, Josh Triplett said that it would be nice to know which kernels will be receiving long-term support ahead of time. The problem with advance notification, Greg said, is that "then people merge a lot of crap" to get it into that release. It was suggested that the announcement could be made after the -rc1 release, making the premature merging of unready code impossible, but, in the end, a few weeks additional notice is not going to change things much. In the end, he said, distributors who are wondering about which kernel will get long-term support should just talk to him. 

Linus, jumping in for the first time this day, let it be know that he dislikes developers who try to game the timing of long-term kernels. Rather than wait until code is ready or fix it on time, they rely on the stable kernels to get fixes out to users eventually. It is, he said, just not good development to do things that way. 

The session wound down with a question: rather than use stable kernels, should companies put current mainline kernels into their products? Olof Johansson responded that he has tried that and "has some battle scars" to prove it. He has seen regressions with mainline kernels, especially on older hardware, which tends not to see much mainline testing. Ben Herrenschmidt added that subsystems like multipath I/O often regress; upgrades cause things to break. Linus said that that sort of attitude tends to perpetuate the problem. This unwillingness to risk problems with mainline kernels is unlikely to change, though, so the stable kernel series is going to prove useful for some time yet. 

**Next** : [The state of linux-next](<https://lwn.net/Articles/608933/>).
