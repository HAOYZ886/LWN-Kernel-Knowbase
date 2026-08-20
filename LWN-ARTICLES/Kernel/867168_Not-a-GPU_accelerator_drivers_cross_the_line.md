---
title: "Not-a-GPU accelerator drivers cross the line"
url: https://lwn.net/Articles/867168/
date: "August 26, 2021"
category: "Device drivers-Accelerators"
author: "By Jonathan Corbet August 26, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 26, 2021

As a general rule, the kernel community is happy to merge working device drivers without much concern for the availability of any associated user-space code. What happens in user space is beyond the kernel's concern and unaffected by the kernel's license. There is an exception, though, in the form of drivers for graphical processors (GPUs), which cannot be merged in the absence of a working, freely-licensed user-space component. The question of which drivers are subject to that rule has come up a few times in recent years; that discussion has now come to a decision point with an effort to block [some Habana Labs driver updates](<https://lwn.net/ml/linux-kernel/20210819110209.GA115485@ogabbay-vm2.habana-labs.com/>) from entry into the 5.15 kernel. 

The GPU-driver rule is the result of a "[line in the sand](<https://lwn.net/Articles/394702/>)" drawn by direct-rendering (DRM) maintainer Dave Airlie in 2010\. The kernel side of most GPU drivers is a simple conduit between user space and the device; it implements something similar to a network connection. The real complexity of these drivers is in the user-space component, which uses the kernel-provided channel to control the GPU via a (usually) proprietary protocol. The DRM maintainers have long taken the position that, without a working user-space implementation, they are unable to judge, maintain, or test the kernel portion of the driver. They have held firm for over a decade now, and feel that this policy is an important part of the progress that this subsystem has made over that time. 

At its core, a GPU is an accelerator that is optimized to perform certain types of processing much more quickly than even the fastest CPU can. Graphics was the first domain in which these accelerators found widespread use, but it is certainly not the last. More recently, there has been a developing market in accelerators intended to perform machine-learning tasks; one of those, the [Habana Gaudi](<https://habana.ai/>), is supported by the Linux kernel. 

The merging of the Gaudi driver has raised a number of questions about how non-GPU accelerators should be handled. This driver did not go through the DRM tree and was not held to that subsystem's rules; it went into the mainline kernel while lacking the accompanying user-space piece. That was later rectified (mostly — see below), but the DRM developers were unhappy about a process that, they felt, bypassed the rules they had spent years defending. [Just over one year ago](<https://lwn.net/Articles/821817/>), the arrival of a couple of other accelerator drivers spurred a discussion on whether those drivers should be treated like GPUs or not; no clear conclusions resulted. 

The Habana driver has been the source of a few similar discussions over the last few months, with bursts [in late June](<https://lwn.net/ml/linux-kernel/CAKMK7uFOfoxbD2Z5mb-qHFnUe5rObGKQ6Ygh--HSH9M=9bziGg@mail.gmail.com/>) and [early July](<https://lwn.net/ml/linux-kernel/YOQXBWpo3whVjOyh@phenom.ffwll.local/>). The problem now is an expansion of that driver's capabilities that requires using the kernel's [DMA-BUF](<https://www.kernel.org/doc/html/v5.13/driver-api/dma-buf.html>) and [P2PDMA](<https://lwn.net/Articles/767281/>) subsystems to move data between devices. These subsystems were developed to work with GPU drivers and are clearly seen by some DRM developers as being part of the kernel's GPU API; drivers using them should, by this reasoning, be subject to the GPU subsystem's merging rules. Or, as Airlie [phrased it](<https://lwn.net/ml/linux-kernel/CAPM=9tyT_iouVE2v8J0SMJOLV=pr=QJOm88ud=bht4=5Ms2Y2A@mail.gmail.com/>) in his objection to merging the Gaudi changes: 

> NAK for adding dma-buf or p2p support to this driver in the upstream kernel. There needs to be a hard line between "I-can't-believe-its-not-a-drm-driver" drivers which bypass our userspace requirements, and I consider this the line. 
> 
> This driver was merged into misc on the grounds it wasn't really a drm/gpu driver and so didn't have to accept our userspace rules. 
> 
> Adding dma-buf/p2p support to this driver is showing it really fits the gpu driver model and should be under the drivers/gpu rules since what are most GPUs except accelerators. 

The interesting twist here, as [acknowledged](<https://lwn.net/ml/linux-kernel/YORLTmyoXDtoM9Ta@phenom.ffwll.local/>) by DRM developer Daniel Vetter, is that there is, indeed, a free user-space implementation of the Gaudi driver. What is still _not_ available is the compiler used to generate the instruction streams that actually drive this device. Without the compiler, Vetter said, the available code is ""still useless if you want to actually hack on the driver stack"". He [elaborated](<https://lwn.net/ml/linux-kernel/CAKMK7uHfCbNQJwbXgLC9ibk71kVG7TBK4bfFxzX82ziSgqG9nw@mail.gmail.com/>) further: 

> Can I use the hw how it's intended to be used without it? 
> 
> If the answer is no, then essentially what you're doing with your upstream driver is getting all the benefits of an upstream driver, while upstream gets nothing. We can't use your stack, not as-is. Sure we can use the queue, but we can't actually submit anything interesting. 

Over the course of the discussions, the DRM developers have tried to make it clear that they want _a_ working, free implementation of the user-space side. It does not have to be _the_ code that is shipped to customers, as long as it is sufficient to understand how the driver as a whole works. To some, though, the compiler requirement stretches things a bit far. Habana developer Oded Gabbay has [described](<https://lwn.net/ml/linux-kernel/CAFCwf10_rTYL2Fy6tCRVAUCf4-6_TtcWCv5gEEkGnQ0KxqMUBg@mail.gmail.com/>) the DRM subsystem's requirements this way: 

> I do think the dri-devel merge criteria is very extreme, and effectively drives-out many AI accelerator companies that want to contribute to the kernel but can't/won't open their software IP and patents. 
> 
> I think the expectation from AI startups (who are 90% of the deep learning field) to cooperate outside of company boundaries is not realistic, especially on the user-side, where the real IP of the company resides. 

Cooperating outside of company boundaries is, of course, at the core of the Linux kernel development process. The DRM subsystem is not alone in making such requirements; Vetter [responded](<https://lwn.net/ml/linux-kernel/CAKMK7uEAJZUHNLreBB839BZOfnTGNU4rCx-0k55+67Nbxtdx3A@mail.gmail.com/>) by pointing out, among other things, that the kernel community will not accept a new CPU architecture without a working, free compiler. 

Over the years, there has been no shortage of problems with vendors that want their hardware to work with Linux while keeping their "intellectual property" to themselves. This barrier has been overcome many times, resulting in wider and better hardware support in the kernel we all use. Getting there has required at least two things: demand from customers for free drivers and a strong position in the development community against proprietary drivers. The demand side must develop on it own (and often does), but the kernel community has worked hard to maintain and communicate a unified position on driver code; consider, for example, the [position statement](<https://lwn.net/Articles/287056/>) published in 2008. As a result there is a consensus in the community covering a number of areas relevant to proprietary drivers; one will have to work hard to find voices in favor of exporting symbols to benefit such drivers, for example. 

This ongoing series of discussions makes it clear that the kernel community has not yet reached a consensus when it comes to the requirements for drivers for accelerator devices. That creates a situation where code that is subject to one set of rules if merged via the DRM subsystem can avoid those rules by taking another path into the kernel. That, of course, will make it hard for the rules to stand at all. Concern about this prospect extends beyond the DRM community; media developer Laurent Pinchart [wrote](<https://lwn.net/ml/linux-kernel/YSLYklow2ZAPqgK1@pendragon.ideasonboard.com/>): 

> I can't emphasize strongly enough how much effort it took to start getting vendors on board, and the situation is still fragile at best. If we now send a message that all of this can be bypassed by merging code that ignores all rules in drivers/misc/, it would be ten years of completely wasted work. 

Avoiding that outcome will require getting kernel developers and (especially) subsystem maintainers to come to some sort of agreement — always a challenging task. 

In the case of the Gaudi driver, Greg Kroah-Hartman [replied](<https://lwn.net/ml/linux-kernel/YR6PIvwYbYG20ZY0@kroah.com/>) that he had pulled the controversial code into his tree. In response to the subsequent objections, he [dropped that work](<https://lwn.net/ml/linux-kernel/YR99zh43GD3AylRY@kroah.com/>) and promised to ""write more"" once time allows. Dropping the patches for now helps to calm the situation, but it has not resolved the underlying disagreement. At some point, the kernel community will have to reach some sort of conclusion regarding its rules for accelerator drivers. Failing that, we are likely to see a steady stream of not-a-GPU drivers finding their way into the kernel — and a lot of unhappiness in their wake.
