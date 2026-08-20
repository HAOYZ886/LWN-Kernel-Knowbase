---
title: "ACPI for ARM?"
url: https://lwn.net/Articles/574439/
date: "November 22, 2013"
category: "ACPI; Architectures-Arm"
author: "By Jonathan Corbet November 22, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 22, 2013

The "Advanced Configuration and Power Interface" (ACPI) was not an obvious win when support for it was first merged into the mainline kernel. The standard was new, actual implementations were unreliable, and supporting it involved bringing a large virtual machine into the kernel. For years, booting with ACPI disabled was the first response to a wide range of problems; one can still find web sites advising readers to do that. But, for the most part, ACPI has settled in as a mandatory part of the PC platform standard. Now, however, it appears that a similar story may be about to play out in the ARM world. 

#### Arguments for and against ACPI

There have been rumblings for a few years that ACPI would start to appear in ARM-based systems, and in server systems in particular. Recently, some code to support such systems has started to make the rounds; Olof Johansson, a co-maintainer of the arm-soc tree, looked at this code and [didn't like what he saw](<https://lwn.net/Articles/574442/>): 

The more I start to see early UEFI/ACPI code, the more I am certain that we want none of that crap in the kernel. It's making things considerably messier, while we're already very busy trying to convert everything over and enable DT -- we'll be preempting that effort just to add even more boilerplate everywhere and total progress will be hurt. 

In this message and several followups Olof clarified what he was trying to get across. The ARM world already has a mechanism to describe the hardware — device trees — that is only now coming into focus. Adding device tree support has required making changes to a large amount of platform and driver code; supporting ACPI threatens to bring just as much work and add a second code path for system configuration that will need to be maintained forever. Even worse is the fact that there are no established standards for ACPI in the ARM setting; nobody really knows how things are supposed to work, and what is coming out in the early stages is not encouraging. Bringing in ARM ACPI support now would be committing the kernel community to supporting a moving target indefinitely. 

Olof [went on to suggest](<https://lwn.net/Articles/574443/>) that it might be best to wait for others to figure out how ACPI on ARM is supposed to work: 

Oh wait, there's people who have been doing this for years. Microsoft. They should be the ones driving this and taking the pain for it. Once the platform is enabled for their needs, we'll sort it out at our end. After all, that has worked reasonably well for x86 platforms. 

He added that, until there are ACPI systems shipping with Windows and working well, the Linux community should stay far away from ACPI on ARM. If ACPI-based systems actually hit the market, he said, they can be supported with a pre-boot layer that translates the system's ACPI tables into the device tree format. 

Disagreement with this position came in a couple of forms. Several people point out that standards developed by Microsoft may not suit the Linux community as well as we might like. As Mark Rutland (a device tree bindings maintainer) [put it](<https://lwn.net/Articles/574444/>): 

I'm not sure it's entirely reasonable to assume that Microsoft will swoop in and develop standards that are useful to us or even applicable to the vast majority of the systems that are likely to exist. If they do, then we will (by the expectation that Linux should be able to run wherever another OS can) have to support whatever standards they may create. 

Russell King [added](<https://lwn.net/Articles/574447/>) another point echoed by many: refusing to support ACPI could cost the community its chance to influence (or even control) how the standard evolves. In his words: 

We have a possibility here to define how we'd like ACPI to look: we have the chance to have ACPI properties using the same naming that we already have for DT. 

Shutting the door on ACPI, Russell asserted, would be a move that the community would regret in the long term. 

Jon Masters [joined the conversation](<https://lwn.net/Articles/574449/>) to make the claim that ARM-based servers were committed to the ACPI path, saying ""all of the big boys are going to be using ACPI whether it's liked much or not"". He said that the server space requires a mechanism that has been standardized and set in stone, and that, in his opinion, the device tree abstraction is far too unstable to be usable (a claim that Grant Likely [strongly disagreed with](<https://lwn.net/Articles/574668/>)). Red Hat, Jon [said](<https://lwn.net/Articles/574450/>), is fully behind ACPI on ARM servers for all of the products that it has absolutely not said it will ever offer. Jon's wording, along with his suggestion that everything has already been decided in NDA-protected conference rooms, won him few friends in this discussion, but his point remains: there will be systems using ACPI on the market, and Linux has to deal with them somehow. 

#### What to do

But that still doesn't answer the question of _how_ to deal with them. Arnd Bergmann [suggested](<https://lwn.net/Articles/574452/>) that ACPI might not be a long-term issue for the ARM community: 

I think we can still treat ACPI on ARM64 as a beginner's mistake and provide hand-written DT blobs for the few systems that start shipping with that. The main reason for doing it in the first place was the expected number of Windows RT servers, but WinRT isn't doing well at the moment, so it's not unreasonable to assume it's going the same way as WinRT tablets. 

Most people, though, seemed to think that ACPI could be here to stay, so the community will have to figure out some way of dealing with it. 

One possibility might be Olof's idea of translating the ACPI tables into a device tree, but that approach was somewhat unpopular. It looks to many like a partial answer to the problem that would run into no end of problems; there is also the matter of running the ACPI Machine Language (AML) code found in the ACPI firmware. AML can be necessary for hardware initialization and power management tasks, but it has no analog in the device tree world. Generally, there was a sentiment that, if ACPI is to be supported on ARM systems, it should be supported properly and not behind some sort of translation layer. 

In the short term, some sort of translation to device trees — either at boot-time or done by hand — seems likely to be the outcome, though. Putting code into the kernel to support any ACPI-based systems that might appear in the near future just seems to many like a way to take on a long-term support burden for short-lived systems. What might start to tip the balance could be systems which, as Arnd [described them](<https://lwn.net/Articles/574454/>), are ""PCs with their x86 CPU removed and an ARM chip put in there""; adding ACPI support for those would be ""harmless enough"", he said. But Arnd seems to be strongly against adding ACPI support for complicated ARM-style systems. 

Longer-term, the community is likely to watch and wait. Efforts will be made to direct the evolution of ACPI for ARM systems; Linaro, in particular, has developers engaged with that process now. And even Olof is [open](<https://lwn.net/Articles/574455/>) to bringing in ACPI support at some point in the future, once its supporters ""seem to have their act together, have worked out their kinks and reached a usable stable platform"". But that, he says, could be a couple of years from now. 

Microsoft, through its dominance of the market for software on PC-class systems, was able to push hardware standards in directions it liked. In the ARM world, Linux dominates just as strongly, so it seems a bit surprising to be playing catch-up with shifts in the ARM platform in this way. Part of the problem, of course, is that there is no single Linux voice at the standards table; companies like Linaro and Red Hat are working on the problem, but they do not represent, or seemingly even talk to, the rest of the community on this topic. The fact that much of this work is done under non-disclosure agreements does not help; NDAs do not fit well with how community development is done. 

In the end, it will certainly work out; it is hard to imagine any significant class of ARM-based hardware being successful without solid Linux support. It's mostly a matter of how much short- and long-term pain will have to be endured to make that support happen. For all the early complaining, ACPI has mostly worked out in the x86 world; it may well find a useful role in the ARM market as well.
