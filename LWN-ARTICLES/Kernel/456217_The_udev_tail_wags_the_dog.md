---
title: The udev tail wags the dog
url: https://lwn.net/Articles/456217/
date: "August 24, 2011"
category: "Development model-User-space ABI; Device tree; udev"
author: "By Jonathan Corbet August 24, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 24, 2011

It is not unheard of for kernel developers to refuse to support a particular user-space interface that, they think, is poorly designed or hard to maintain into the future. A user-space project refusing to use a kernel-provided interface in the hope of forcing the creation of something better is a rather less common event. That is exactly what is happening with the udev project's approach to device tree information, though; the result could be a rethinking of how that information gets to applications. 

OLPC laptops have, among their other quirks, a special keyboard which requires the loading of a specific keymap to operate properly. For the older generations of laptops, loading this keymap has been easily handled with a udev rule: 

```
ENV{DMI_VENDOR}=="OLPC", ATTR{[dmi/id]product_name}=="XO", \
    		RUN+="keymap $name olpc-xo"
```

This rule simply extracts the name of the machine from the desktop management interface (DMI) data that has been made available in sysfs. If that data indicates that the system is running on an XO laptop, the appropriate keymap file is loaded. DMI is an x86-specific interface, though, and the upcoming (1.75) generation of the XO laptop is not an x86-based machine. There is no DMI information available on that laptop, so this rule fails; some other solution will be needed. 

In current times, the source for hardware description information - especially on non-discoverable platforms - is supposed to be the device tree structure. So [Paul Fox's solution](<https://lwn.net/Articles/456221/>) would seem to make sense: he created a new rule with a helper script to extract the machine identification from the device tree, which happens to be available in `/proc/device-tree`. It almost certainly came as a surprise when this solution was [rejected](<https://lwn.net/Articles/456222/>) by udev maintainer Kay Sievers, who said: 

Reading such things from /proc is kinda taboo from code we maintain in udev. All things not related to processes really do not belong into /proc and udev code should never get into the way of possibly deprecating these things in the long run, even when they might never happen. I know, there is sometimes no other simple option, but we generally prefer the inconvenience it causes, over adding hacks to upstream code, to make a move to a generally useful solution (which isn't /proc/*) more attractive. 

Of course, Paul wasn't adding the `/proc/device-tree` interface; criticism of such a move would not have been surprising. That file has a long history; it has been supported, under some architectures, since the 2.2 kernel. So one might think that it is a bit late to be complaining about it; there are a number of `/proc` files added in those days which would not be allowed into `/proc` now. In general, those files are considered to be part of the user-space ABI at this point; like it or not, we are stuck with them. The device tree file has been around for long enough that it almost certainly falls into that category; it's hard to imagine that it would have been maintained for so long if there were no programs making use of it. Whether or not the udev developers like it, `/proc/device-tree` is not likely to go anywhere else anytime soon. 

That still doesn't mean that the udev developers have to make use of it, though, and they seem determined to hold out for something better. [Quoting Kay](<https://lwn.net/Articles/456226/>) again: 

No, sorry, the time for dirty hacks in userspace, and work-arounds for architectures and platforms that don't provide what is commonly used elsewhere is over. There is no rush, it's new functionality, and no need to start with 'transitions periods' that in reality will never end. Stuff just needs to be fixed properly these days, and papering over just hurts us more in the end. 

Kay would like to see the machine identification information exposed separately somewhere under `/sys`; it has even been suggested that platforms using device trees could emulate the DMI directory found on x86 systems. That, to them, looks like a longer-term solution that doesn't put udev in the position of blocking an ABI cleanup sometime in the future. 

In essence, what we have is a user-space utility telling the kernel that an interface it has supported for well over a decade is unacceptable and needs to be replaced. To force that replacement, udev is refusing to accept changes that make use of the existing interface. Whether that is a proper course of action depends on one's perspective. To some, it will look like a petty attempt to force kernel developers to maintain two interfaces with duplicate information in the hope that a long-lived `/proc` file will eventually go away, despite its long history. To others, it will seem like a straightforward attempt to help the kernel move toward interfaces that are more supportable in the long term. 

In this particular case, it looks like udev probably wins. Adding the machine identification somewhere in sysfs will be easy enough that it is probably not worth the effort to fight the battle. In a more general sense, this episode shows that the kernel ABI is not just something handed down to user space from On High. User-space developers will have their say, even a dozen years after the interface has been established; that is a good thing. Having more developers look at these issues from both sides of the boundary can only help in the creation of better interfaces.
