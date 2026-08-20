---
title: In a bind with binder
url: https://lwn.net/Articles/618421/
date: "October 29, 2014"
category: Android
author: "By Jonathan Corbet October 29, 2014 Linux
Plumbers Conference"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 29, 2014

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2014-Linux_Plumbers_Conference>)

The Android microconference at the 2014 Linux Plumbers Conference started off with an assessment of how the Android project was doing with regard to integration of its kernel changes with the mainline. The general feeling was that things are getting better; with 3.14, Android carries "only" 346 patches on top of the mainline kernel. But subsequent events have shown that the long-lasting tension between Android and mainline kernel developers has not entirely dissipated yet; the much-maligned Android "binder" mechanism is now one of the central points of that tension. 

Many of the Android-specific components that _are_ in the mainline kernel still live in the staging tree. Early in the session, Greg Kroah-Hartman got up to discuss whether any of those components should be moved out of staging — either out of the kernel tree entirely, or into the mainline proper. As an example of the first type, the Android "logger" module, which is no longer used as of the Lollipop release, may simply go away entirely. The story with binder is different, though. 

Android's binder has a long history, having first shown up as part of the BeOS system. It is a mechanism for remote procedure calls and remote object management that is used heavily within the Android system. For an overview of how binder relates to other interprocess communication mechanisms, see [this 2011 article](<https://lwn.net/Articles/466304/>) by Neil Brown. Almost nobody seems to like binder; it is seen as abusing various low-level kernel interfaces, has known security problems when used outside of the tightly controlled Android setting, and more. But Android needs it, so it persists. 

In the Plumbers session, Greg noted that binder had been in the staging tree for years. Since it is in active use, binder is not going away anytime soon. It is "horrible" and "broken by design," but it is an ABI [![\[Greg
Kroah-Hartman\]](https://static.lwn.net/images/conf/2014/lce-lpc/GregKroahHartman-sm.jpg)](<https://lwn.net/Articles/618457/>) that we need to support, Greg said, so we might as well move it out of the staging tree. No objections to the idea were raised in the session; everybody seemed happy with the idea of getting binder out of staging. 

Greg wasted no time before [posting a patch](<https://lwn.net/Articles/618424/>) to move binder out of staging; it was on the lists before the Android microconference had concluded its business. Whether Greg expected the wider discussion to go as smoothly as the microconference is not clear; what is clear is that's not what he got. 

John Stultz, who has done a lot of work toward the mainlining of Android-specific features, [expressed](<https://lwn.net/Articles/618425/>) a few concerns. The first of those had to do with maintenance: who was going to be the maintainer of this code, and had the Android developers agreed with that decision? Greg's [response](<https://lwn.net/Articles/618426/>) to this question was to note that the binder code had not changed in any significant way for a long time; there is not, he feels, a lot of maintenance that is needed. To the extent that binder needs a maintainer, Greg has volunteered to do it. 

Another concern of John's had to do with efforts to replace binder with something better tied into the kernel; work is progressing on writing a binder-compatible library over the (still out-of-tree) [kdbus](<https://lwn.net/Articles/580194/>) interface. Moving binder into the mainline, he thought, might reduce the incentive to get that work done. Greg's answer here was that any such work is entirely new code; it doesn't mitigate the need to maintain the existing code "forever." ""So as there really is nothing left to do on it, it deserves to be in the main part of the kernel source tree."" 

Finally, John worried that moving binder to the mainline might encourage others to make use of it; this was a concern that Alan Cox [shared](<https://lwn.net/Articles/618428/>). Greg's response here is that there is never a way to control how others will use the software we ship. But, if anybody outside of Android were to use binder, he said, ""you deserve all of the pain and suffering and rooted machines you will get."" 

Arnd Bergmann raised [a number of issues](<https://lwn.net/Articles/618429/>) mostly relating to security. Evidently Android does not use the full API exported by binder; he would like to see an audit of how the API is used so that the unneeded parts can be removed, reducing the attack surface of the whole thing. Binder also leaks kernel-space pointers into user space and has no awareness of namespaces, so it can also leak information between containers in undesirable ways. These points have not been addressed in the discussion so far. 

Finally, Christoph Hellwig [attempted to block the move outright](<https://lwn.net/Articles/618449/>), saying: 

NAK. It's complete rubbish and does things to the FD code that it really shouldn't. Android needs to completely redo the interface, and there's been absolutely no work towards that. 

Greg [disagreed](<https://lwn.net/Articles/618451/>) about the claim that no work was being done toward a new interface; he also repeated that, no matter how that work goes, the existing interface needs to be supported indefinitely. Christoph was not satisfied by the answer, though; here he represents a group of kernel developers who feel that the Android developers are still not really trying to work with the mainline kernel. From this point of view, merging substandard Android code just allows Google to offload the pain of maintaining it and encourages more of the same behavior in the future. About the only tool that the kernel developers have to address this problem, as they see it, is their ability to refuse to accept inadequate code. 

At that point, the discussion wound down. Greg has not said whether he still plans to move binder regardless of the points that have been raised. In the end, it could be said to make little difference; binder will be shipped with the kernel tree whether it is moved or not. But the decision to move binder or not could send a message on how the kernel development community feels about its relationship with the Android team. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Linux Plumbers Conference].
