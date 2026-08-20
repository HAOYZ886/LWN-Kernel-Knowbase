---
title: Broadcom firmware and regulatory compliance
url: https://lwn.net/Articles/406491/
date: "September 22, 2010"
category: "Device drivers-Wireless networking; Networking-Wireless"
author: "By Jonathan Corbet September 22, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 22, 2010

Broadcom's recently-announced open source wireless networking driver was a major step forward for a company which has, until now, not been forthcoming when it comes to free support for its wireless products. That driver includes the obligatory firmware blob which has been licensed for free distribution by the company; it is now found in the kernel firmware repository. Broadcom has not freed the firmware for its older drivers, though, leading to discussions on the intersection between kernel development and regulatory compliance. 

The lack of freely-distributable firmware for older Broadcom products makes life a bit more difficult for users, who must obtain the firmware separately. When the new firmware was made distributable, David Woodhouse asked the company about the older firmware as well, only to told that it would not be made distributable. As he [explained](<http://lwn.net/Articles/405376/>) it, Broadcom is afraid that allowing the distribution of that firmware could lead to trouble: 

They seem to think that they could be prosecuted even for *enabling* people to use the open source b43 driver, because you have the possibility of hacking that driver not to conform to the regulatory requirements. 

The reason why the old firmware is different is simple: the newer firmware, which can only run on newer hardware, has regulatory compliance built into it. The older firmware, instead, depends on the driver in the kernel to ensure that it is not configured to operate in a non-compliant manner. 

David is not known for graceful suffering in the presence of (people he sees as) fools. His response was [a patch](<https://lwn.net/Articles/406493/>) which "credits" Broadcom for enabling the development of the reverse-engineered b43 driver; this "enablement" is said to have come through the provision of binary-only drivers which could be reverse engineered. His goal in writing this patch was described as: 

Everything we do in the b43 and b43legacy drivers is enabled by Broadcom's original binary-only drivers. So let's make that 'enablement' by Broadcom's binary drivers clear in our source code -- in the hope that it'll narrow the 'risk gap' that they falsely perceive between open and closed source drivers. 

Or failing that, in the hope that it'll give their crack-addled lawyers aneurysms, and they'll hire some saner ones to replace them. 

He also expressed a wish that the b43 developers would release more information - obtained from the binary-only drivers - on how to patch those binary drivers to get around various regulatory restrictions. Once again, he feels that this kind of information would help to make it clear that free drivers do not make it any easier to operate the hardware in an illegal manner. 

David's position plays well with developers who have no patience for obstacles created by lawyers. There is also a vocal contingent out there which says that Linux has no business telling users how they should use their hardware in any case; if the user wants to configure the hardware in a non-compliant manner, that's the user's problem. In some cases, that user may well have a license which makes it entirely legal to run the hardware outside of the parameters which normally apply to off-the-shelf wireless networking equipment. So regulatory compliance naturally irritates developers who think that the kernel has no business getting in anybody's way in this regard. 

Luis Rodriguez, on the other hand, is a strong supporter of regulatory compliance in the Linux kernel; he [stepped into the discussion](<https://lwn.net/Articles/406494/>) to remind people of the [kernel's regulatory statement](<http://wireless.kernel.org/en/developers/Regulatory/statement>) and to say that there was no real interest in encouraging the violation of spectrum-use regulations with any driver. He [added](<https://lwn.net/Articles/406495/>): 

The reason why current legislation doesn't seem to make sense is because it does not, but just because a law doesn't make sense it does not enable vendors to ignore it. So the best you can do in the meantime is really be proactive by working on real technical solutions. 

We are not dealing with legal issues on Linux, we are dealing with engineering solutions, and trust me, we're light years ahead of other OSes because of this now. 

His point is that the kernel's "engineering solution" to the regulatory problem has made it possible for wireless vendors to dip their toes into the open-source water. That, in turn, has helped to move Linux from having poor wireless support to, arguably, having the best support over the course of a few years. It is hard to argue with the success which the wireless developers have had recently; any moves which might endanger that success should be considered carefully, to say the least. 

Of course, it would be nicer to do without the proprietary firmware blob altogether. In early 2009, the openfwwf project [announced](<http://lwn.net/Articles/314313/>) the availability of an open source firmware implementation for Broadcom adapters. Since then, news from that project has been relatively scarce. On September 21, though, Michael Busch [announced](<http://lwn.net/Articles/406379/>) the availability of a toolchain for working with the b43 firmware. Using the disassembler and assembler, it is possible to decode the device firmware, make changes, then build a new firmware load. Naturally, one can also build a new firmware implementation from the beginning. With these tools available, we might just get to a point where we can have device firmware without distribution restrictions, and which adds features and flexibility to the device as well.
