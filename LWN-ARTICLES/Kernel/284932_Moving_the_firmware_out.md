---
title: Moving the firmware out
url: https://lwn.net/Articles/284932/
date: "June 4, 2008"
category: Firmware
author: "By Jake Edge June 4, 2008"
---

By **Jake Edge**  
June 4, 2008

It seems that David Woodhouse had a bit of an ulterior motive when he recently [reworked the kernel firmware loader](<http://lwn.net/Articles/284104/>). That is not to say the work is not useful in its own right, but one of his goals is more apparent now: removing all of the firmware from the kernel source tree. By making it easy to separate the firmware blobs--while still allowing them to be statically built into kernels--he has provided a possible path for all firmware needed by any Linux driver to live in a single place. 

The firmware issue is somewhat contentious, with licensing and political issues that tend to annoy the kernel developers. Arguments about the "legality" of distributing firmware with the kernel flare up from time to time. Separate from that, there are some good reasons why it makes sense to keep the firmware in its own place: some distributions need or want to distribute their kernels without firmware blobs and some hardware manufacturers will not allow their firmware to be distributed with the kernel because of concerns about the GPL. The current situation makes it harder for both users and distributors. 

Woodhouse brought up the idea of pulling the firmware out of the kernel in a [post to linux-kernel and ksummit-2008-discuss](<https://lwn.net/Articles/284967/>). The agenda for this year's Kernel Summit is under discussion, so he proposed that it be discussed there. He is clearly trying to anticipate the technical concerns that others might have: 

By the time the kernel summit comes around, we should have made decent progress on moving _all_ the firmware blobs to the firmware/ directory. And at that point I'd like to remove them completely, to a separate git tree and tarball. Those who really want to build them in to their static kernel would still be able to, but it wouldn't be the default behaviour. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Unsurprisingly, there are some fairly strenuous objections. David Miller [is quite annoyed](<https://lwn.net/Articles/284985/>): 

Sorry, that's taking things too far. I've fought, like, forever, to keep the tg3 driver with it's firmware in-tree. I refuse to let the driver get broken like that, it's staying working, and that means in-tree and linked into the driver. 

If debian or whoever else have these concerns and want to rip the firmware out, it is one hundred percent their problem to patch things out of the kernel tree they use. 

But there are other reasons to collect firmware in one single place, as Arjan van de Ven [notes](<https://lwn.net/Articles/284988/>): 

Right now it's a royal pain for users to get all the right pieces of firmware.... having ONE place to put all that would go a long way of making that side of things easier. 

If you want to argue that that should be in the kernel tarball itself, you won't hear me complain. But others will... and for that a 2nd tarball might well be the answer. Just we shouldn't need 100 tarballs.

There is a very real concern, though, that putting firmware without source into the kernel is a GPL violation. It is impossible to know for sure without a court decision, which is something that no one wants to have to deal with. Companies--and their lawyers--tend to be very conservative when it comes to inviting lawsuits, so removing unrelated, possibly actionable code from the kernel sources is of great benefit to them. As Woodhouse [says](<https://lwn.net/Articles/284990/>): 

And it isn't just the nutters. Fedora also wants to ship the firmware in a separate package from the kernel -- since the alleged GPL violation is such a _gratuitous_ risk given that we always use an initrd anyway, and because people want to be able to do 'Free' spins which don't feature the firmware at all, even in the source packages. 

By making it easier to put all of the firmware in one non-GPL tree, hardware vendors--and _their_ lawyers--may be willing to allow the firmware to be distributed. If Woodhouse's plan for supporting both compile-time and runtime loading of the firmware is successful and reasonably transparent, there should be little difference for kernel developers, but big improvements for users and distributors. It is unclear whether this is something that will be resolved in email, as Woodhouse hopes, or will require a discussion at the Kernel Summit in September, but it's an idea with a lot of merit that may find its way into the mainline at some point.
