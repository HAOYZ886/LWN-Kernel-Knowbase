---
title: Kernel configuration for distributions
url: https://lwn.net/Articles/507239/
date: "July 18, 2012"
category: "Build system-Kernel configuration; Distributions"
author: "By Jake Edge July 18, 2012"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
July 18, 2012

Configuring a kernel was once a fairly straightforward process, only requiring knowledge of what hardware needs to be supported. Over time, things have gotten more complex in general, but distributions have added their own sets of dependencies on specific kernel features--dependencies that can be difficult for regular users to figure out. That led Linus Torvalds to put out an [RFC proposal](<https://lwn.net/Articles/507276/>) to add distribution-specific kernel configuration options. 

The problem stems from distributions' user space needing certain configuration options enabled in order to function correctly. Things like tmpfs and devtmpfs support, control groups, security options (e.g. SELinux, AppArmor), and even raw netfilter table support were listed by Torvalds as "support infrastructure" options that are required by various distributions. But, in addition to being hard to figure out, those options tend to change over time, so a configuration that worked for Fedora N may not work for Fedora N+1. The resulting problems can be hard to find as Torvalds pointed out: ""There's been several times when I started with my old minimal config, and the resulting kernel would boot, but something wouldn't quite work right, and it can be very subtle indeed."" 

So, he suggested adding distribution-specific Kconfig files: 

The point I'm slowly getting to is that I would actually love to have *distro* Kconfig-files, where the distribution would be able to say "These are the minimums I *require* to work". So we'd have a "Distro" submenu, where you could pick the distro(s) you use, and then pick which release [...] it would make it much easier for a normal user (and quite frankly, I want to put myself in that group too) to make a kernel config that "just works". 

There are others ways to get there, of course, but they leave something to be desired, Torvalds said. Copying the distribution config file would work, but would bring along a bunch of extra options that aren't really necessary for the proper operation of the distribution. Using `make localmodconfig` (which selects all of options from the running kernel) suffers from much the same problem, he said. The ultimate goal is to have more people able to build kernels: 

I really think that "How do I generate a kernel config file" is one of those things that keeps normal people from compiling their own kernel. And we *want* people to compile their own kernel so that they can help with things like bisecting etc. The more, the merrier. 

In general, the idea was met with approval on linux-kernel. There were concerns about how the distribution-specific files would be maintained, and that sometimes they might get out of sync with the distribution's requirements. Dave Jones [noted](<https://lwn.net/Articles/507314/>) that he sometimes gets blindsided by Fedora kernel requirements (and he is the Fedora kernel maintainer). 

Torvalds is pretty explicitly [not looking for a perfect solution](<https://lwn.net/Articles/507317/>), however, just one that is better: ""even a 'educated guess' config file is better than what we have now"". In that message, he outlines two requirements that he sees for the feature. The first is that each configuration option that is selected for a particular distribution version come with a comment explaining why it is needed. The second is that the configuration options be the minimum required to make the system function properly--not that it ""grow to contain all the options just because somebody decided to just add random things until things worked"". 

Commenting the options may be difficult even for those who work directly on distribution kernels though. Ben Hutchings (who maintains the Debian kernel) [pointed out](<https://lwn.net/Articles/507321/>) that he sometimes does not know the reason that a particular option is needed, particularly at some later point: ""just because an option was requested and enabled to support some bit of userland, doesn't mean I know what's using or depending on it now"". 

Other kinds of configuration options are possible, of course. In his original message, Torvalds mentioned configurations for "common platforms", such as a "modern PC laptop" that would choose options typically required for those (USB storage, FAT/VFAT, power management, etc.). He specifically said that platform configuration should be considered an entirely separate feature from the distribution idea. 

KVM (and other virtualization) users were also interested in creating an option that would select all of the drivers and other options needed for those kernels. Currently ""you need to hunt through 30+ different menus in order to find what you need to run in a basic KVM virtual machine"", as Trond Myklebust [put it](<https://lwn.net/Articles/507332/>). There was a lot of discussion (and much agreement) on the need for better configuration options for virtualization, but some of that got rather far afield from Torvalds's original proposal. 

Unsurprisingly, kernel developers started thinking about how _they_ could use the feature. There was concern that choosing a particular distribution and its dependencies would make it harder for kernel developers to further customize the configuration. David Lang had some [specific complaints](<https://lwn.net/Articles/507338/>) about the approach suggested in the RFC, noting that it would be hard to choose a Fedora kernel without getting SELinux for example. He also was concerned about the amount of churn these defconfig-like files might cause (referencing the movement to reduce the number of defconfigs in the ARM tree). But Torvalds [makes it clear](<https://lwn.net/Articles/507339/>) that Lang and other kernel hackers are not the target of the feature: 

The thing I'm asking for is for normal people. Make it easy for people who DO NOT CARE about the config file to just build a kernel for their machine. 

Don't complicate the issue by bringing up some totally unrelated question. Don't derail a useful feature for the 99% because you're not in it. 

There may be ways to satisfy both camps--Lang seemed to think so anyway--but until someone actually posts some code, it's hard to say. While there was general agreement that the feature would be useful, so far no one has stepped up to do the work. Whether Torvalds plans to do that or was just floating a trial balloon and hoping someone else would run with it is unclear, but it does seem like a feature worth having.
