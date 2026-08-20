---
title: Dracut looks to replace the initramfs patchwork
url: https://lwn.net/Articles/317793/
date: "February 4, 2009"
category: Dracut; initramfs
author: "By Jake Edge February 4, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
February 4, 2009

Creating initramfs images, for use by the kernel at "early boot" time, is a rather messy business. It is made more so by the fact that each individual distribution has its own tools to build the image, as well as its own set of tools inside it. At the 2008 Kernel Summit, Dave Jones spent some time [discussing the problem](<https://lwn.net/Articles/298593/>) along with his idea to start over by creating a cross-distribution initramfs. That has led to the Dracut project, which was [announced](<http://lwn.net/Articles/311890/>) by Jeremy Katz in December, and a new [mailing list](<http://vger.kernel.org/vger-lists.html#initramfs>), aptly named "initramfs", in which to discuss it. 

An initramfs is a cpio archive of the initial filesystem that gets loaded into memory when the kernel is loaded. That filesystem needs to contain all of the drivers and tools needed to mount the real root filesystem. It isn't strictly necessary to have an initramfs, a minimal `/dev` along with the required drivers built into the kernel is another alternative. Distributions, though, all use an initramfs and, over time, each has come up with their own way to handle this process. Jones, Katz, and others would like to see something more standardized, that gets pushed upstream into the mainline kernel so that distributions can stop fussing with the problem. 

There are a number of advantages to that approach. Building an initramfs from the kernel sources would eliminate problems that users who build their own kernels sometimes run into. If a distribution's initramfs scheme falls behind the pace of kernel development in some fashion, users can find themselves unable to build a kernel+initramfs combination that will work. There is also hope that dracut will help speed up the boot process by using udev, as Katz [puts it](<https://lwn.net/Articles/317872/>): 

By instead moving to where we're basing everything off of uevents we can hopefully move away from the massive shell scripts of doom, speed up boot and also maybe get to where a more general initramfs can be built _with the kernel_ instead of per-system. 

Because initramfs is so integral to the early boot process--and so difficult to debug if problems arise--there is a concern about starting over. It is not surprising, then, that there is some resistance to throwing out years of hard-earned knowledge that is embodied in the various distributions' initramfs handling, leading Maximilian Attems to [ask](<https://lwn.net/Articles/317873/>): 

btw why do we need dracut at all? your blog has vague allusion against initramfs-tools, which is much better tested and has seen the field. 

beside having more features and flexibility it does not hardcode udev usage, nor bash, why should it not be considered at first!? 

It is a question that is frequently asked, but one that Jones has a ready [answer](<https://lwn.net/Articles/317874/>) for: 

"why not use the ubuntu one?"  
"why not use the suse one?" 

they all have some good and bad tradeoffs. Distro X has feature Y which no-one else does. etc. 

When the project began we spent some time looking at what everyone else already does, and "lets start over and hope others participate" seemed more attractive than taking an existing one and bending it to fit. 

So, the Red Hat folks, at least, are proceeding with dracut. Jones recently posted a [status report](<http://www.codemonkey.org.uk/2009/02/02/dracut-status/>) on his blog that outlined what is working and what still needs to be done. Though it currently is ""Fedora-centric, with a few hardcoded assumptions in there, so it'll likely fall over on other distros"", fixing that is clearly high on the to-do list. The status report is an effort to get people up-to-speed so that other distributions can start trying it out. In addition, he plans to start trying it on various distributions himself. 

In its current form, dracut is rather minimal. It has a script named `dracut` that will generate a gzipped cpio file for the initramfs image, as well as an `init` shell script that ends up in that image. Jones says that `init` ""achieves quite a lot in its 119 lines"": setting up device nodes, starting udev, waiting for the root device to show up and mounting it, mounting `/proc` and `/sys`, and more. If anything goes wrong during that process, `init` will drop to a shell that will allow diagnosis of the problem. So far, it only supports the simpler cases for the location of the root filesystem: 

Currently, dracut supports root on raw disks (/dev/sda), lvm (/dev/mapper...), and mounting root by label or uuid. If you have a more esoteric rootfs setup, such as root-on-nfs, right now it'll fail horribly. 

There is only one remaining barrier to getting rid of the unlamented `nash`, and that is a utility to do a switch_root (i.e. switch to a new root directory and start an `init` from there). The plan is to write a standalone utility that would be added to the `util-linux` package. The environment provided by the initramfs would include `util-linux`, `bash`, and use `glibc`, which doesn't sit well with some embedded folks. They generally prefer a statically linked `busybox` environment. Kay Sievers [outlines](<https://lwn.net/Articles/317885/>) the reasons for a standard environment: 

Busybox is nice as an option to be able to rescue/hack. It should definitely be provided as an optional "plugin" for people who need it. But there is no chance to depend on it by default, for the very same reason klibc, or any other libc is not an option. 

Full-featured distros who make their money with support, can just not afford to support tools compiled differently from the tools in the real rootfs. SUSE used klibc for one release, and stopped doing that immediately, because you go crazy if you run into problems with bootup problems on [customer] setups you can not reproduce with the tools from the real rootfs. 

There is plenty to do to make dracut into a real tool for creating initramfs images--at least ones that work on more than just Fedora--more root filesystem types need to be handled, hibernation signatures need to be recognized and handled, the udev rules need to be cleaned up, kdump images need to be supported, etc. But the overriding question is: will other distributions start working on dracut as well? If and when Jones (or others) get things at least limping along on Debian/Ubuntu and/or SUSE, will those distributions start getting on board? So far, there is not a lot of evidence of anyone other than Red Hat working on dracut. 

But, the plan is to eventually submit dracut upstream to the mainline kernel, so that `make initramfs` works in a standard kernel tree. It would seem that many kernel hackers see the need for standardizing initramfs and eventually moving it into the kernel, as Ted Ts'o [notes](<https://lwn.net/Articles/317894/>): 

[...] So the idea that was explored was adding a common mkinitramfs with basic functionality into kernel sources, with the ability for distributions to add various "value add" enhancements if they like. This way if the kernel wants to move more functionality (for example, in the area of resuming from hibernation) out of the kernel into initramfs, it can do so without breaking the ability of older distributions from being able to use kernel.org kernels. 

So IMHO, it's important not only that the distributions standardize on a single initramfs framework, but that framework get integrated into the kernel sources. 

No one is very happy about losing their particular version of the tools to build an initramfs--if only because of familiarity--but a standardized solution is something whose time has come. Probably any of the existing tools could have been used as a starting point, but for [political reasons](<http://kernelslacker.livejournal.com/2008/09/22/>), it makes sense to start anew. There is a fair amount of cruft that has built up in the existing tools as well, which folks are unlikely to miss, so there are also technical reasons to start over. It should come as no surprise that a project started by Red Hat might be somewhat Fedora-centric in its early form, but the clear intent is to make it distribution-agnostic. It would seem the right time for other distributions and constituencies (embedded for example) to get involved to help shape dracut into something useful for all.
