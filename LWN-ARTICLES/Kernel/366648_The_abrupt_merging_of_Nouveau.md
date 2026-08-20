---
title: The abrupt merging of Nouveau
url: https://lwn.net/Articles/366648/
date: "December 15, 2009"
category: "Device drivers-Nouveau; Nouveau"
author: "By Jonathan Corbet December 15, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 15, 2009

The merge window is normally a bit of a hectic time for subsystem maintainers. They have two weeks in which to pull together a well-formed tree containing all of the changes destined for the next kernel development cycle. Occasionally, though, last-minute snags can make the merge window even more busy than usual. The unexpected merging of the [Nouveau driver](<http://nouveau.freedesktop.org/wiki/>) is the result of one such snag - but it is a story with a happy ending for all. 

Dave Airlie probably thought he had enough on his plate when he generated the [DRM pull request](<https://lwn.net/Articles/366650/>) for 2.6.33. This tree contained 203 commits touching 122 different files, and adding over 9,000 lines of code. One of the key features aimed at the kernel is the new "page flipping `ioctl()`," helpfully described in the commit message as ""The ioctl takes an fb ID and a ctrc ID and flips the crtc to the given fb at the next vblank."" In English, it means that a specific video output can be quickly switched from one region of video memory to another, allowing for clean video changes without the "tearing" that results from display of a video buffer which is being changed. 

Other changes for DRM this time around include support for Intel's "Ironlake" GPU and "Pineview" Atom processor, and a great deal of work supporting kernel mode setting on Radeon GPUs. Radeon, it seems, only lacks good power management support at this point; it will likely lose its "staging" designation before the end of this development cycle. 

Linus was not impressed by any of that, though. Instead, he had [one concern](<https://lwn.net/Articles/366653/>): the fact that the Nouveau driver \- a reverse-engineered driver for NVIDIA chipsets - was not a part of the pull request. Nouveau had been discussed at the [2009 Kernel Summit](<https://lwn.net/Articles/KernelSummit2009/>), and it was generally agreed that this code should find its way into the mainline as soon as possible. 2.6.33 is the first merge window since the summit, and Linus clearly had expected some action on that front. When he didn't get it, he made his disappointment known. 

One might wonder what the problem with Nouveau was. The world is full of out-of-tree Linux drivers; recent efforts have reduced their number considerably, but they still exist and Linus does not normally complain about them. Certainly Nouveau has a higher profile than most other out-of-tree drivers; it is the only hope for a free driver for a large percentage of available machines. But the real problem is that Fedora (at least) has been shipping this driver without doing enough (in Linus's opinion) to get it upstream. In [Linus's words](<https://lwn.net/Articles/366655/>): 

I'm pissed off at distribution people. For years now, distributions have talked about "upstream first", because of the disaster and fragmentation that was Linux-2.4. And most of them do it, and have been fairly good about it. 

But not only is Fedora not following the rules, I know that Fedora people are actively making excuses about not following the rules. I know Red Hat actually employs (full-time or part-time I have no idea) some Nouveau developer, and by that point Red Hat should also man up and admit that they need to make "merge upstream" be a priority for them. 

A number of reasons for the non-merging of Nouveau have been given, ranging from "not ready yet" and "unstable user-space API" to "we haven't found the time yet." The real blocker in recent times, though, has been the binary blob loaded into some NVIDIA GPUs by the driver. This chunk of code, known as the "voodoo" or "ctxprogs," was obtained by watching the proprietary drivers in action. Since nobody in the Nouveau project wrote this code, nobody has been willing to sign off on it; it's not at all clear that it can be legally distributed. Linus has [not been impressed](<https://lwn.net/Articles/366657/>) by this reason either, but the fact remains: developers take the `Signed-off-by:` line seriously and are not willing to attach it to something which might be legally questionable. 

The obvious answer, one which has been applied in other situations, is to pull the firmware out of the driver and load it into the kernel at run time. And that is exactly what happened with Nouveau: Ben Skeggs put in an intensive effort to remove ctxprogs and use the firmware loading API to get it when the driver loads. Dave then put together the ["DRM Nouveau pony tree"](<http://lwn.net/Articles/366205/>) and requested that it be pulled for 2.6.33. Linus, of course, did exactly that. 

Potential users will still have to get the "ctxprogs" from elsewhere. For whatever reason, pointers to "elsewhere" are hard to find, but your editor happens to know that the firmware can be found in [the Nouveau git tree](<http://cgit.freedesktop.org/nouveau/linux-2.6/tree/firmware/nouveau>). Simply grabbing the right version and placing it in the local firmware directory should be sufficient. 

All of this marks significant progress for Nouveau, but a dependence on firmware of dubious origin is likely to inhibit the adoption of this driver in the long term. So it was good to learn (via [an LWN comment posting](<http://lwn.net/Articles/366266/>)) that the contents of the ctxprogs blob are not quite as obscure as many of us had thought: 

[W]e know a lot about ctxprogs these days, including their purpose [context switching], what they do [save/restore PGRAPH state], and most of their opcodes. There are still some unknowns that prevent us from writing new ctxprogs from scratch right now, but we're working on that and it *will* be resolved in the proper way. Which is throwing out nvidia's progs and writing our own prog generator. 

It seems that things are moving quickly on this front too; on December 15, Ben [announced](<https://lwn.net/Articles/366660/>) the availability of a replacement firmware for NVIDIA GeForce 6/7 hardware. This is a first posting for this code; doubtless testers will encounter some problems. But it sounds very much like the hardest problems have been overcome, at least for this particular variant of the hardware. With luck, NVIDIA's firmware will not be needed for much longer. In the longer term, it might even turn out to be possible to program interesting functions into the hardware, extending its capabilities in surprising ways. 

Once upon a time, Linux users had to be very careful about which hardware they bought. Over the years, most of those problems have gone away; it is now easy to find systems which are completely supported by free software. One of the biggest exceptions has been in the area of graphics. Vendors like Intel and ATI/AMD have made the decision that their hardware should be supported with free drivers ([most of the time](<http://lwn.net/Articles/366276/>)) and have invested resources to make that happen. NVIDIA has been rather less cooperative, and support for its hardware has suffered accordingly. It would appear that the driver problem is getting close to a solution, but we should never forget the effort which was required to get to this point. NVIDIA would be far more worthy of our future commercial support if it had not made that effort necessary.
