---
title: Nouveau and interface compatibility
url: https://lwn.net/Articles/377953/
date: "March 10, 2010"
category: "Development model-User-space ABI; Device drivers-Nouveau"
author: "By Jake Edge March 10, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
March 10, 2010

A recent linux-kernel discussion, which descended into flames at times, took on the question of the stability of user-space interfaces. The proximate cause was a change in the interface for the Nouveau drivers for NVIDIA graphics hardware, but the real issues go deeper than that. Though the policy for the main kernel is that user-space interfaces live "forever", the policy in the staging tree has generally been looser. But some, including Linus Torvalds, believe that staging drivers that have been shipped by major distributions should be held to a higher standard. 

As part of the just-completed 2.6.34 merge window, Torvalds pulled from the DRM tree at Dave Airlie's request, but immediately ran into [problems](<https://lwn.net/Articles/378014/>) on his Fedora 12 system: 

Hmm. What the hell am I supposed to do about 

```
(II) NOUVEAU(0): [drm] nouveau interface version: 0.0.16
    	(EE) NOUVEAU(0): [drm] wrong version, expecting 0.0.15
```

The problem stemmed from the Nouveau driver changing its interface, which required an upgrade to `libdrm`--an upgrade that didn't exist for Fedora 12. The Nouveau changes have been backported into the Fedora 13 2.6.33 kernel, which comes with a new `libdrm`, but there are no plans to put that kernel into Fedora 12. Users that stick with Fedora kernels upgraded via `yum` won't run into the problem as Airlie [explains](<https://lwn.net/Articles/378020/>): 

At the moment in Fedora we deal with this for our users, we have dependencies between userspace and kernel space and we upgrade the bits when they upgrade the kernels, its a pain in the ass, but its what we accepted we needed to do to get nouveau in front of people. We are currently maintain 3 nouveau APIs across F11, F12 and F13. 

That makes it impossible to test newer kernels on Fedora 12 systems with NVIDIA graphics, though, which reduces the number of people who are able to test. In addition, there is no "forward compatibility" either--the kernel and DRM library must upgrade (or downgrade) in lockstep. Torvalds is concerned about losing testers who run Fedora 12, as well as problems for those on Fedora 13 (Rawhide right now) who might need to bisect a kernel bug--going back and forth across the interface-change barrier is not possible, at least easily. In his original complaint, Torvalds is characteristically blunt: ""Flag days aren't acceptable."" 

The Nouveau drivers were only [merged for 2.6.33](<http://lwn.net/Articles/366648/>) at Torvalds's request--or demand--and they were put into the staging tree. The staging tree configuration option clearly spells out the instability of user-space interfaces: ""Please note that these drivers are under heavy development, may or may not work, and may contain userspace interfaces that most likely will be changed in the near future."". So several kernel hackers were clearly confused by Torvalds's outburst. Jesse Barnes [put it](<https://lwn.net/Articles/378026/>) this way: 

Whoa, so breaking ABI in staging drivers isn't ok? Lots of other staging drivers are shipped by distros with compatible userspaces, but I thought the whole point of staging was to fix up ABIs before they became mainstream and had backwards compat guarantees, meaning that breakage was to be expected? 

Yes, it sucks, but what else should the nouveau developers have done? They didn't want to push nouveau into mainline because they weren't happy with the ABI yet, but it ended up getting pushed anyway as a staging driver at your request, and now they're stuck? Sorry this whole thing is a bit of a wtf... 

But Torvalds doesn't disagree that the interface needs changing, he is just unhappy with the way it was done. Because the newer `libdrm` is not available for Fedora 12, he can't [test it](<https://lwn.net/Articles/378032/>): 

I'm not going to release a kernel that I can't test. So if I can't get a libdrm that works in my F12 environment, I will _have_ to revert that patch that you asked me to merge. 

It is not just Torvalds who can't test it, of course, so he would like to see something done that will enable Fedora users to test and bisect kernels. The Nouveau developers don't want to maintain multiple interfaces, and the Fedora (and other distribution) developers don't want to have to test multiple versions of the DRM library. As Red Hat's Nouveau developer Ben Skeggs [put it](<https://lwn.net/Articles/378034/>): ""we have no intention of keeping crusty APIs around when they aren't what we require."" 

Torvalds would like to see a way for the various `libdrm`s to co-exist, preferably with the X server choosing the right one at runtime. As he [notes](<https://lwn.net/Articles/378042/>), the server has the information and, if multiple libraries are installed, the right one is only a `dlopen()` away: 

Who was the less-than-rocket-scientist that decided that the right thing to do was to "check the kernel DRM version support, and exit with an error if it doesn't match"? 

See what I'm saying? What I care about is that right now, it's impossible to switch kernels on a particular setup. That makes it effectively impossible to test new kernels sanely. And that really is a _technical_ problem. 

In the end, Airlie helped him [get](<https://lwn.net/Articles/378047/>) both of the proper libraries installed on his system, with a symbolic link to (manually) choose between them. That was enough to allow testing of the kernel, thus Torvalds didn't revert the Nouveau patch in question. But there is a larger question here: When should a user-space interface be allowed to change, and, just how should it be done? 

The Nouveau developers seem rather unhappy that Torvalds and others are trying to change their development model, at least partially because they never requested that Nouveau be merged. But Torvalds is not really pushing the Nouveau developers so much as he is pushing the distributor who shipped Nouveau to handle these kinds of problems. In his opinion, once a major distributor has shipped a library/kernel combination that worked, it is responsible for ensuring that it continues to work, especially for those who might want to run newer kernels. 

The problem for testers exists because the distribution, in this case Fedora, shipped the driver before getting it into the upstream kernel, which violates the "upstream first" principle. Torvalds [makes it clear](<https://lwn.net/Articles/378052/>) that merging the code didn't cause the problem, shipping it did: 

So the watershed moment was _never_ the "Linus merged it". The watershed moment was always "Fedora started shipping it". That's when the problems with a standard upstream kernel started. 

Alan Cox [disagrees](<https://lwn.net/Articles/378053/>), even quoting Torvalds from 2004 back at himself, because the Nouveau developers are just developing the way they always have; it's not their fault that the code was shipped and is now upstream: 

Someone who never made a commitment to stability decided to do the logical thing. They deleted all the old broken interfaces, they cleaned up their ioctls numbering and they tided up afterwards. I read it as the action of someone who simply doesnt acknowledge that you have a right to control their development and is continuing to work in the way they intended. 

But the consensus, at least among those who aren't graphics driver developers, seems to be that user-space interfaces should only be phased out gradually. That gives users and distributions plenty of time to gracefully handle the interface change. That is essentially how mainline interface changes are done; even though user-space interfaces are supposed to be maintained forever, they sometimes do change--after a long deprecation period. In fact, Ingo Molnar [claimed](<https://lwn.net/Articles/378049/>) that breaking an ABI often leads to projects that either die on the vine or do not achieve the success that they could: 

I have _never_ seen a situation where in hindsight breaking the ABI of a widely deployed project could be considered 'good', for just about any sane definition of 'good'. 

It's really that simple IMO. There's very few unconditional rules in OSS, but this is one of them. 

Ted Ts'o [sees](<http://lwn.net/Articles/378055/>) handling interface changes gracefully as part of being a conscientious member of the community. If developers don't want to work that way, they shouldn't get their code included into distributions: 

You say you don't want to do that? Then keep it to your self and don't get it dropped into popular distributions like Fedora or Ubuntu. You want a larger pool of testers? Great! The price you need to pay for that is to be able to do some kind of of ABI versioning so that you don't have "drop dead flag days". 

Had this occurred with a different driver, say for an obscure WiFi device, it is likely there would have been less, or no, outcry. Because X is such an important, visible part of a user's experience, as well as an essential tool for testers, breaking it is difficult to hide. Torvalds has always pushed for more testing of the latest mainline kernels, so it shouldn't come as a huge surprise that he was less than happy with what happened here. 

This situation has cropped up in various guises along the way. While developers would like to believe they can control when an ABI falls under the compatibility guarantee, that really is almost never the case. Once the interface gets merged, and user space starts to use it, there will be pressure to maintain it. It makes for a more difficult development environment in some ways, but the benefit for users is large.
