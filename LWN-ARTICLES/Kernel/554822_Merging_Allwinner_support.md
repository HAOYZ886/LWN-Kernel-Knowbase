---
title: Merging Allwinner support
url: https://lwn.net/Articles/554822/
date: "June 19, 2013"
category: "Architectures-Arm; Development model-Driver merging"
author: "By Jake Edge June 19, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 19, 2013

Getting support for their ARM system-on-chip (SoC) families into the mainline kernel has generally been a goal for the various SoC vendors, but there are exceptions. One of those, perhaps, is Allwinner Technology, which makes an SoC popular in tablets. Allwinner seems to have been uninterested in the switch to Device Tree (DT) in the mainline ARM kernel (and the requirement to use it for new SoCs added to the kernel tree). But the story becomes a bit murkier because it turns out that developers in the community have been doing the work to get fully DT-ready support for the company's A1X SoCs into the mainline. While Allwinner is not fully participating in that effort, at least yet, a recent call to action with regard to support for the hardware seems to be somewhat off-kilter. 

The topic came up in response to a [note](<https://lwn.net/Articles/554821/>) from Ben Hutchings on the debian-release mailing list (among others) that was not specifically about Allwinner SoCs at all; it was, instead, about his disappointment with the progress in the Debian ARM tree. Luke Leighton, who is acting as a, perhaps self-appointed, "go-between" for the kernel and Allwinner, [replied](<https://lwn.net/Articles/554913/>) at length, noting that the company would likely not be pushing its code upstream: 

well, the point is: the expectation of the linux kernel developers is that Everyone Must Convert To DT. implicitly behind that is, i believe, an expectation that if you *don't* convert to Device Tree, you can kiss upstream submission goodbye. and, in allwinner's case, that's simply not going to happen. 

As might be guessed, that didn't sit well with the Linux ARM crowd. ARM maintainer Russell King had a [sharply worded response](<https://lwn.net/Articles/554922/>) that attributed the problem directly to Allwinner. He suggested that, instead of going off and doing its own thing with "fex" (which serves many of the same roles that DT does in the mainline), the company could have pitched in and helped fix any deficiencies in DT. In addition, he is skeptical of the argument that DT was not ready when Allwinner needed it: 

DT has been well defined for many many years before we started using it on ARM. It has been used for years on both PowerPC and Sparc architectures to describe their hardware, and all of the DT infrastructure was already present in the kernel. 

Leighton, though, [points](<https://lwn.net/Articles/554927/>) to the success of the Allwinner SoCs, as well as the ability for less-technical customers to easily reconfigure the kernel using fex as reasons behind the decision. There are, evidently, a lot of tablet vendors who have limited technical know-how, so not having to understand DT or how to transform it for the bootloader is a major plus: 

the ODMs can take virtually any device, from any customer, regardless of the design, put *one* [unmodified, precompiled] boot0, boot1, u-boot and kernel onto it, prepare the script.fex easily when the customer has been struggling on how to start that DOS editor he heard about 20 years ago, and boot the device up, put it into a special mode where the SD/MMC card becomes a JTAG+RS232 and see what's up... all without even removing any screws. 

The discussion continued in that vein, with ARM kernel developers stating that the way forward was to support DT while Leighton insisted that Allwinner would just continue to carry its patches in its own tree and that Linux (and its users) would ultimately lose out because of it. Except for one small problem: as Thomas Petazzoni [pointed out](<https://lwn.net/Articles/554975/>), Maxime Ripard has been working on support for the Allwinner A1X SoCs—merged into the 3.8 kernel in `arch/arm/mach-sunxi`. 

In fact, it turns out that Ripard has been in contact with Allwinner and gotten data sheets and evaluation boards from it. He [pointed](<https://lwn.net/Articles/554983/>) Leighton to a [wiki](<http://linux-sunxi.org/Linux_mainlining_effort>) that is tracking the progress of the effort. That work has evidently been done on a volunteer basis, as Ripard is interested in seeing mainline support for those SoCs. 

In the end, Leighton's messages start to degenerate into what might seem like an elaborate troll evidencing a serious misunderstanding of how Linux kernel development happens. In any case, he seems to think he is in a position to influence Allwinner's management to pursue an upstream course, rather than its current development path. But his [demands](<https://lwn.net/Articles/554999/>) and his [suggestion](<https://lwn.net/Articles/555004/>) that he apologize on behalf of the Linux kernel community for ""not consulting with you (allwinner) on the decision to only accept device tree"" elicited both amazement and anger—for obvious reasons. 

Leighton appears to start with the assumption that the Linux kernel and its community _need_ to support Allwinner SoCs, and that they need to beg Allwinner to come inside the tent. It is a common starting point for successful silicon vendors, but time and again has been shown to not be the case at all. In fact, Allwinner's customers are probably already putting pressure on the company to get its code upstream so that they aren't tied to whichever devices and peripherals are supported in the Allwinner tree. 

As far as fex goes, several in thread suggested that some kind of translator could be written to produce DT from fex input. That way, customers who want to use a Windows editor to configure their device will just need to run the tool, which could put the resulting flattened DT file into the proper place in the firmware. Very little would change for the customers, but they would immediately have access to the latest Linux kernel with its associated drivers and core kernel improvements. 

Alternatively, Allwinner could try to make a technical case for the superiority of fex over DT, as Russell King [suggested](<https://lwn.net/Articles/555026/>). It seems unlikely to be successful, as several developers in the thread indicated that it was a less-general solution than DT, but it could be tried. Lastly, there is nothing stopping Allwinner from continuing down its current path. If its customers are happy with the kernels it provides, and it is happy to carry its code out of tree, there is no "Linux cabal" that will try force a change. 

Evidently, though, that may not actually be what Allwinner wants. Its efforts to support Ripard's work, along with contacts made by [Olof Johansson](<https://lwn.net/Articles/555037/>), [Ripard](<https://lwn.net/Articles/555039/>), and others, indicate that Allwinner is interested in heading toward mainline. It essentially started out where many vendors do, but, again like many SoC makers before it, decided that it makes sense to start working with upstream. 

We have seen this particular story play out numerous times before—though typically with fewer comedic interludes. In a lot of ways, it is the _vendors_ who benefit most from collaborating with the mainline. It may take a while to actually see that, but most SoC makers end up there eventually—just as with other hardware vendors. There are simply too many benefits to being in the mainline to stay out of tree forever.
