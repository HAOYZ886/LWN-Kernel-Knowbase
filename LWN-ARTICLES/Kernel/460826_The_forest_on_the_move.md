---
title: The forest on the move
url: https://lwn.net/Articles/460826/
date: "September 28, 2011"
category: "Development tools-Infrastructure"
author: "By Jonathan Corbet September 28, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 28, 2011

As of this writing, kernel.org remains offline, though it is to be hoped that access for git trees, at least, will be restored before too long. Linus's current plans seem to involve opening the merge window before mid-October; without a functioning kernel.org, that will not run anywhere near as smoothly as the community might like. Still, some things cannot be rushed, and it is important that kernel.org come back in a solid and secure mode. 

Quite a few trees have found new homes in the mean time. Here is an updated version of the list of relocated trees: 

> ACPI| https://github.com/lenb/linux.git  
> ---|---  
> ALSA| git://github.com/tiwai/sound.git  
> ALSA driver| git://github.com/tiwai/alsa-driver-build.git  
> ALSA SOC| git://opensource.wolfsonmicro.com/linux-2.6-asoc.git  
> amd64 EDAC| git://amd64.org/linux/bp.git  
> APM| git://twin.jikos.cz/jikos/apm  
> arm-soc| git://git.linaro.org/people/arnd/arm-soc.git  
> DRM| git://people.freedesktop.org/~airlied/linux  
> fbdev| https://github.com/schandinat/linux-2.6  
> HID| git://twin.jikos.cz/jikos/hid  
> hwspinlock| git://github.com/ohadbc/hwspinlock-next.git  
> infiniband|  https://github.com/rolandd/infiniband  
> input| https://github.com/dtor/input  
> ipvs| git://github.com/horms/ipvs.git  
> kbuild| http://repo.or.cz/w/linux-kbuild.git  
> kvm| git://github.com/avikivity/kvm.git  
> libata| git://github.com/jgarzik/libata-dev.git  
> linux-next| git://github.com/sfrothwell/linux-next.git  
> mainline| git://github.com/torvalds/linux.git  
> mmc| git://dev.laptop.org/users/cjb/mmc mmc-next  
> networking| git://github.com/davem330/net  
> pm| git://github.com/rjwysocki/linux-pm.git  
> regmap| git://opensource.wolfsonmicro.com/regmap.git  
> SCSI| git://bedivere.hansenpartnership.com/git/scsi-rc-fixes-2.6.git  
>  git://bedivere.hansenpartnership.com/git/scsi-misc-2.6.git  
> slab| git://github.com/penberg/linux.git  
> tip| git://tesla.tglx.de/git/linux-2.6-tip  
> tmem| git://oss.oracle.com/git/djm/tmem.git  
> trivial| git://twin.jikos.cz/jikos/trivial  
> utrace| git://github.com/utrace/linux.git  
> v9fs| git://github.com/ericvh/linux.git  
> wireless| git://git.infradead.org/users/linville/wireless.git  
>  git://git.infradead.org/users/linville/wireless-next.git  
>  git://git.infradead.org/users/linville/wireless-testing.git  
> xen| git://oss.oracle.com/git/kwilk/xen.git  
  
That is a substantial list of moved trees, but, as linux-next maintainer Stephen Rothwell [noted](<https://lwn.net/Articles/460618/>) on September 27, that leaves 89 trees which still only exist on kernel.org. Those trees will not have seen any updates since kernel.org went off the net. Some of them will certainly be trees that are currently idle or close to it; not every tree feeding into linux-next carries patches for every development cycle. But others presumably exist for a reason; if kernel.org does not come back soon, they will need to find a different home. 

One significant tree that has not moved is the stable release tree; the last stable updates came out on August 29\. 

With luck, kernel.org will come back soon and the above list will become moot. But kernel.org, when it returns, may look somewhat different. It has already been announced that there will be no shell access to the machines hosting the git trees. There may be other security measures put into place as well, some possibly requiring changes in how developers operate. Making changes of that nature in the time left before the next merge window could be hard to do. The 3.2 development cycle, in other words, might be a bit more interesting and less smooth than usual.
