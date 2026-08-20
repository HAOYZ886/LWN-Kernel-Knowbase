---
title: "The intersection of modules, GKI, and rocket science"
url: https://lwn.net/Articles/872209/
date: "October 11, 2021"
category: "Android-Generic kernel image; Development model-Loadable modules; Modules"
author: "By Jonathan Corbet October 11, 2021"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 11, 2021

One does not normally expect a lot of controversy around a patch series that makes changes to platform-specific configurations and drivers. The furor over some work on the Samsung Exynos platform may thus be surprising. When one looks into the discussion, things become more clear; it mostly has to do with disagreements over the best ways to get hardware vendors to cooperate with the kernel development community. 

In mid-September, Will McVicker posted [a brief series](<https://lwn.net/ml/linux-kernel/20210920190350.3860821-1-willmcvicker@google.com/>) of changes for the Exynos configuration files; one week later, [a larger, updated series](<https://lwn.net/ml/linux-kernel/20210928235635.1348330-1-willmcvicker@google.com/>) followed. The purpose in both cases was to change a number of low-level system-on-chip (SoC) drivers to allow them to be built as loadable modules. That would seem like an uncontroversial change; it is normally expected that device drivers will be modular. But the situation is a little different for these low-level SoC drivers. 

#### Generic kernels and essential drivers

In the distant past, kernels for Arm-based SoCs were specific to the target platform. While x86 kernels normally run on all x86 processors, Arm kernels were built for a small range of target platforms and would not boot on anything else. Over the years, the Arm developers have worked to make the 64-bit Arm kernel sufficiently generic that a single binary image can boot on a wide range of platforms. To a great extent, this portability has been achieved by building drivers as modules so that a kernel running on a given device need only load the drivers that are relevant to that device. 

There is a bootstrapping problem to solve, though; before the kernel can load a single module, it must be able to boot to a point where it has the platform in a known, stable state and is able to mount a RAM-based root filesystem. That can only happen if the drivers needed to boot that far are built into the kernel itself. Thus, the generic kernel contains a long list of platform-specific drivers to configure clocks, pin controllers, and more; without them, the kernel would never boot. The maintainers' policy has long stated that any drivers which are essential for the boot process must be built into the kernel itself. 

McVicker's patch set takes a number of these essential drivers and, for reasons to be discussed in detail below, removes them from the kernel image, making them into loadable modules instead. Ostensibly, this change does not violate the policy for these drivers, but only if it can be demonstrated that the drivers are, in fact, not essential for the kernel to boot on the affected hardware. Therein lies the first big problem for this patch set: McVicker made it clear in the cover letter that these changes had not actually been tested on the appropriate hardware. While he is optimistic that the systems should still boot with modular drivers, nobody has yet proved that. 

Optimism only gets one so far in the kernel community. This lack of testing has caused Exynos platform maintainer Krzysztof Kozlowski to repeatedly push back on the patches; until he is sure that all Exynos systems can boot with those drivers as modules, he is unwilling to take the changes. He has [offered](<https://lwn.net/ml/linux-kernel/db808138-e839-48d8-26e7-659cc9950163@canonical.com/>) to help with some of the needed testing. Meanwhile, Arnd Bergmann has [backed up](<https://lwn.net/ml/linux-kernel/CAK8P3a0zezKvexqvL29Oc44uQq-8QG7LwZy31VYJuYAYbh-Utw@mail.gmail.com/>) Kozlowski's reticence: 

> The "correctness-first" principle is not up for negotiation. If you are uncomfortable with the code or the amount of testing because you think it breaks something, you should reject the patches. Moving core platform functionality is fundamentally hard and it can go wrong in all possible ways where it used to work by accident because the init order was fixed. 

The lack of testing seems like the kind of problem that should be amenable to a solution. Reaching the needed level of confidence may take a while, though. Some systems running a given SoC may boot without a specific clock driver (say) because the firmware initializes the clocks to a reasonable configuration at power-on. Counting on all firmware to have its act together in this way can be a risky endeavor, though. Even so, this testing, which should have been done before the patches were ever submitted, should be possible to fill in after the fact. 

#### Out-of-tree code

There is still the question of _why_ one would want to make this possibly risky change. The obvious benefit is making the core kernel image smaller; this is especially appreciated on all of the platforms that _don't_ use the drivers in question and thus see them as dead weight. But there is another motivation here that relates to a different kernel, also called "generic". 

The kernels shipped on Android devices have notoriously contained vast amounts of out-of-tree code, to the point that such code sometimes outweighs the mainline code that is in use on the device. This has led to problems throughout the ecosystem, including a lack of cooperation with upstream kernel developers, the fragmentation of the Android kernel space, the inability to apply security updates when the vendor inevitably stops doing so, and the cost of maintaining all of those kernels. To address this problem, Google has been pushing vendors of Android-based devices toward its "generic kernel image" (GKI), which is a core kernel that must be shipped by all Android devices. Vendors are able to supply their own modules to load into that kernel, but they cannot replace the kernel itself. 

This policy brings a number of benefits. Vendor changes are restricted to what can be done with the module API, and Google has been pushing to restrict that somewhat as well. The days of vendors replacing the CPU scheduler should be done now. Vendors, naturally, chafe at these restrictions, but they have little alternative to compliance. If they choose to run their own system, even if it is an Android fork, they lose access to many of the Google apps and services that make Android useful for their customers. 

Code that is built into the GKI kernel thus cannot be changed by device vendors. Code that is loaded from modules, instead, can be shoved aside and replaced. Viewed in this light, the desire to modularize built-in drivers becomes rather easier to understand. Even so, there are two different aspects of this situation that are worth examining. One is that vendors want to ship out-of-tree modules on their devices rather than upstreaming their drivers to hide their secret magic from competitors. As Lee Jones [described it](<https://lwn.net/ml/linux-kernel/YVWCK5QO331rfhJJ@google.com/>): 

> In order for vendors to work more closely with upstream, they need the ability to over-ride a *few* drivers to supplement them with some functionality which they believe provides them with a competitive edge (I think you called this "value-add" before) prior to the release of a device. This is a requirement that cannot be worked around. 

As one might imagine, this position is seen as less than fully compelling by much of the kernel development community. It is also not entirely convincing; as Tomasz Figa [put it](<https://lwn.net/ml/linux-kernel/CA+Ln22FBy2ks9gX3df=rQw-6W3iftMVoqsoqBPchGCqDDoMaLg@mail.gmail.com/>): 

> Generally, the subsystems being mentioned here are so basic (clock, pinctrl, rtc), that I really can't imagine what kind of rocket science one might want to hide for competitive reasons. 

Jones [tried to de-emphasize](<https://lwn.net/ml/linux-kernel/YVb%2FpQ1l34TcP81G@google.com/>) this point of discussion later on, but it was a bit late; he had said (part of) the quiet part out loud. 

The other piece of the puzzle is simpler to understand. Even if a set of clock drivers contains no real secrets of interest, the vendor may simply lack the desire to make the effort to get the drivers upstream. It is possible to get out-of-tree drivers built into the GKI, but Google would clearly rather not deal with that anymore, so there is a continual pressure to get drivers into the mainline. If the drivers can be supplied directly by the vendor as a module, instead, they disappear from the GKI and that pressure vanishes. With regard to the Exynos changes, a lack of desire to work upstream seems like a plausible explanation; as Kozlowski pointed out in the above-linked message, Samsung has only contributed a single change to the Exynos subsystem since 2017. 

Jones has [tried to characterize](<https://lwn.net/ml/linux-kernel/YUx1c%2FxZ+nP5aI+X@google.com/>) vendors' upstream reticence as temporary, saying ""vendors are not able to upstream all functionality right away"". Later, though, he [said](<https://lwn.net/ml/linux-kernel/YUyMyVezyjfv1Hs7@google.com/>): 

> But [they have] no incentive to upstream code [for] old (dead) platforms that they no longer make money from. We're not talking about kind-hearted individuals here. These are business entities. 

If neither new or old code can be upstreamed, then it would appear that mainline support for these platforms is at a dead end. 

#### Better or worse?

The natural reaction for many kernel developers is to make life harder for vendors that are seemingly looking for ways to avoid engaging with the development community. That would include rejecting the patch set under consideration here. Olof Johansson's [opinion](<https://lwn.net/ml/linux-kernel/CAOesGMgSt_mYvRzF0rC=fnjMYGO9EX0_Ow2cD1d8XKLD5pHsZA@mail.gmail.com/>) was: 

> This patchset shouldn't go in. 
> 
> GKI is a fantastic effort, since it finally seems like Google has the backbone to put pressure on the vendors to upstream all their stuff. 
> 
> This patch set dilutes and undermines all of that by opening up a truck-size loophole, reducing the impact of GKI, and overall removes leverage to get vendors to do the right thing. 

McVicker, instead, [argued](<https://lwn.net/ml/linux-kernel/CABYd82b9wmzZHAQwBqn-6uJupEV=r6Zx7fiaz+o4Xxt8RQppUA@mail.gmail.com/>) that modularizing these drivers is a way to bring vendors closer to upstream and will improve the situation overall: 

> We believe that if we make it easier for SoC vendors to directly use the upstream kernel during bring-up and during the development stages of their project, then that will decrease the friction of working with upstream (less downstream changes) and increase the upstream contributions. 

Which of these positions is closer to the truth is hard to say; each may hold water with respect to some vendors while falling down with others. Getting vendors to engage with upstream is a constant process requiring judicious use of both carrots and sticks. 

That said, the outcome of this particular discussion is not in a great deal of doubt. Making life easier for uncooperative vendors is usually not, on its own, sufficient reason to keep a patch set out of the kernel. Bergmann described it well in the above-linked message: 

> I understand that it would be convenient for SoC vendors to never have to upstream their platform code again, and that Android would benefit from this in the short run. 
> 
> From my upstream perspective, this is absolutely a non-goal. If it becomes easier as a side-effect of making the kernel more modular, that's fine. 

So, in a sense, much of the discussion was irrelevant; if the patches can be shown to work properly (which has not yet happened), then they are consistent with many of the community's long-term goals and will likely find their way into the mainline sooner or later. Whether that will encourage vendors to work upstream or, instead, make it easier for them to stay away remains to be seen. But problems with uncooperative vendors have existed for as long as the Linux kernel has; they will not go away regardless of what happens here.
