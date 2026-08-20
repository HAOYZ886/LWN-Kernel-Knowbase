---
title: "KS2007: Hardware support and the i386/x86_64 merger"
url: https://lwn.net/Articles/248400/
date: "September 6, 2007"
category: Device drivers; Nouveau; x86 64
author: "By Jonathan Corbet September 6, 2007 LWN.net Kernel Summit 2007 coverage"
---

By **Jonathan Corbet**  
September 6, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

Dirk Hohndel is an outspoken critic of how the kernel community is currently dealing with uncooperative hardware vendors; his LCE talk on this subject was recently [covered in LWN](<http://lwn.net/Articles/248041/>). As the leader of the session on hardware support, Dirk repeated some of his points, made some new ones, and set the stage for a major announcement. 

The session started with a report from John Linville, the wireless networking stack maintainer. The now-famous reverse-engineered ath5k driver for Atheros chipsets is in progress, though it does not yet work reliably for all hardware. There are, he says, going to be some interesting developments with this software in the near future. 

The only vendor who is putting any sort of development effort into free drivers is Intel, unfortunately. There are occasional process difficulties with Intel's developers, but as a whole, things are going reasonably well. One remaining big issue is regulatory compliance. Until the regulatory situation eases, a number of vendors will continue not to cooperate with the Linux community. 

David Airlie talked about video drivers. The Nouveau project continues to work on a reverse-engineered driver for NVidia chipsets, but progress is very slow. Some chips can run glxgears, most cannot. There is a new memory management module which needs to get into the mainline before a number of tasks (texturing, in particular) can be addressed. On the other hand, 2D support is getting better than the current (free) nv driver, at least some of the time. There are, it seems, a vast number of per-card tweaks which must be applied, and those are still being discovered. The Nouveau driver is probably still a full year away from being usable. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

On the ATI front, the Avivo effort has achieved decent 2D support, but no 3D so far. There is also some conversations happening with Via which could result, someday, in better support for those chipsets. 

AMD manager Chris Schlaeger was then challenged to "give us some good news." According to Chris, Linux is very important to AMD; the company believes that the Opteron processor would not have been anywhere near as successful without Linux support. The company's future plans lead to an interesting problem, though. The "Fusion" product line will feature a central processor and a graphics processor on the same die. Continuing to support free software on the CPU while keeping the GPU closed leads to all sorts of contradictions; it's really not an option. So, to avoid losing Linux support altogether, AMD has made an important decision. 

Starting with the R500 chipset and going forward, AMD will fully support free drivers for all of its graphics processors. This support will _not_ take the form of a release of the current proprietary ATI driver; that code is not considered to be something that anybody would really want to look at. So there will be a clean start. AMD will release specifications and a skeleton driver with the plan to have 2D support working by the end of the year. The company is clearly hoping that the community will do much of the work on the driver, but it also plans to participate actively in the process. If AMD follows through - and there is no real reason to believe that it will not - then driver problems for AMD/ATI chipsets will soon be a distant memory. 

Dirk then stood up to talk about the problems faced by companies which try to work with the community; this talk repeated much of the material from [his LCE presentation](<http://lwn.net/Articles/248041/>). What was different was that, for this audience, Dirk asked the development community to push back harder against recalcitrant vendors. We might not want to ban binary-only modules altogether, but we should increase the amount of pain associated with maintaining those modules. It is time to actively make life hard for binary-only vendors. 

A fear was expressed that such a policy might drive away vendors altogether. Dirk responded that, in the current market, walking away from Linux is no longer an option. Vendors have to work with Linux in one way or another. Another developer suggested that making things harder would mostly succeed in upsetting users. According to Dirk, that is part of creating pain for the vendors; upset users will eventually move to hardware which presents fewer problems. The amount of sympathy for this idea varied; some developers would rather work in making life easier for cooperative companies. The idea of making it easier to integrate drivers into the mainline was raised again. 

In the final moments of this session, Bdale Garbee stated that, increasingly, HP is pushing its suppliers for components which are supported by free software. Intel, too, is doing that. According to Bdale, vendors need to hear one thing clearly: the days of selling closed hardware are coming to an end, soon. Such words were well received in this room, to say the least. 

The next session covered a related topic: how the x86 architecture is to be supported in the future. In particular, the idea of [merging the i386 and x86_64 architecture trees](<http://lwn.net/Articles/243704/>) was on the agenda. Much of the discussion followed the lines of LWN's previous coverage of the topic, so it won't be repeated here. 

It did take some time to cover that ground again, though, before Linus made a pronouncement: the current 32-bit/64-bit split does not work, and he intends to merge the patch joining the two architectures regardless of what maintainer Andi Kleen thinks. Andi replied that this can certainly be done, but that a new maintainer would have to be found for the combined architecture; one hopes that he does not follow through with that statement. Almost everybody else was behind the idea, though, with the PowerPC and S/390 developers talking about what a big win it had been for them. So this merger would appear to be a done deal; the only open question is whether it can be ready in time for the 2.6.24 merge window.
