---
title: "Device drivers and non-disclosure agreements"
url: https://lwn.net/Articles/203562/
date: "October 9, 2006"
category: Development model; Device drivers
author: 
---

Anybody who has been working with free software for any period of time knows that hardware support is often one of the community's thorniest problems. Manufacturers are often reluctant to tell their customers how to actually use the hardware they sell. For some strange reason, people buy that hardware anyway, and promptly want it to work with their operating system of choice. If that system is Windows, the manufacturer will usually provide a driver (of uncertain quality). Free software users, instead, are usually on their own. 

The situation is better now than it often has been in the past; free operating systems support a wide variety of hardware. In many cases, the vendors have given in and simply released programming information required for anybody to write a driver. In many others, however, this information is provided to a specific company or developer under a non-disclosure agreement (NDA), with the understanding that the resulting driver would then be released under a free license. This approach has, beyond a doubt, made more drivers available for use with our systems; it has become a common way of doing things, especially in the Linux world. 

Not everybody is happy with this state of affairs, however. [OpenBSD](<http://openbsd.org/>) founder Theo de Raadt has started a campaign against the practice of writing drivers under NDA; in the process, he has stepped on, if anything, more than the usual number of toes, to the point that some of the people involved are now refusing to talk to him. Theo's tactics are never subtle, but he does have a point which is worth listening to. 

At a first glance, a driver developed under NDA seems like a good thing. It is free software, after all, and it makes the device work under the target operating system. But these drivers can be problematic for the simple reason that they do not document the hardware the way the specification does. Without that documentation, many of the benefits of free software are lost. 

In many cases, only the original author can maintain a driver developed under NDA. Nobody else has the documentation required to make any real changes to how the driver operates; nobody else really understands the device. Whenever a new version of the hardware comes out, or whenever somebody needs a feature that the original author didn't see fit to implement, one can only hope that said author is still around and in a mood to work on that driver. 

This situation can be worse yet if the author who signed the NDA writes poor quality code, full of constants whose meaning is clear to nobody. In some cases, the vendor may require that the driver be written in that way in order to expose as little information about the hardware as possible. It's worth noting that this is a problem associated with poor hardware documentation in general. Your editor recently had cause to dig into the [OmniVision OV7x20 sensor driver](<https://lwn.net/Articles/203563/>). The data sheet for this device can be found by anybody with access to a search engine, but that data sheet is little help for anybody trying to understand this code: 

```
/* Settings for (color) OV7620 camera chip */
        static struct ovcamchip_regvals regvals_init_7620[] = {
    	{ 0x12, 0x80 }, /* reset */
    	{ 0x00, OV7620_DFL_GAIN },
    	{ 0x01, 0x80 },
    	{ 0x02, 0x80 },
    	{ 0x03, OV7620_DFL_SAT },
    	{ 0x06, OV7620_DFL_BRIGHT },
    	{ 0x07, 0x00 },
    	{ 0x0c, 0x24 },
    	{ 0x0c, 0x24 },
    	{ 0x0d, 0x24 },
    	/* ... 45 lines of this stuff removed ... */
    	{ 0x74, 0x00 },
    	{ 0x75, 0x8e },
    	{ 0x76, 0x00 },
    	{ 0x77, 0xff },
    	{ 0x78, 0x80 },
    	{ 0x79, 0x80 },
    	{ 0x7a, 0x80 },
    	{ 0x7b, 0xe2 },
    	{ 0x7c, 0x00 },
    	{ 0xff, 0xff },	/* END MARKER */
        };
```

It's not clear that _anybody_ really knows what all those register settings do; they involve a number of bits and registers which are marked "reserved" in the documentation. For all practical purposes, they constitute a form of opaque firmware which must be loaded into the device for it to operate correctly. Pain will come to anyone who attempts anything more than the most trivial tweaks to these values. 

Similar issues (in an entirely different context) recently led Linus Torvalds to [exclaim](<https://lwn.net/Articles/203610/>): 

And we should tell all hardware companies that firmware tables are stupid, and that we just want to know what the hell the registers MEAN! 

Without complete hardware documentation, we will not understand what our peripherals are doing. 

Finally, a big problem with drivers written under NDA is that they only work on one system, and they can be very little help for anybody trying to make the device work on a different kernel. That, of course, has a lot to do with why there is a lot of criticism of this approach coming from the BSD world while the Linux community tends to be more accepting of it. It is probably safe to say that most developers who are able to get this sort of access to documentation are working on Linux drivers. If we were pounding our heads against our monitors in an attempt to reverse-engineer hardware by way of obscure BSD drivers written under NDA, we might see the situation in a different light. 

Theo has picked out two targets for special attention: Intel and the One Laptop Per Child (OLPC) project. Intel has gotten a fair amount of good press supporting its hardware under Linux. The truth of the matter, however, is that a number of drivers for Intel hardware are written in-house, with little or no hardware documentation provided to the community. As long as Intel remains interested in maintaining those drivers, things will work well enough - for Linux users. BSD users are not so lucky, however, and we may all be out of luck if a change of management or focus at Intel causes the company to drop its Linux drivers. If Intel truly wants to be known as an open-source friendly company, it would do well to make its hardware truly open. The OpenBSD developers are currently [running a campaign](<https://lwn.net/Articles/203569/>) aimed at pushing Intel in that direction.  Disclosure time  
---  
Readers of this article should be aware that your editor is in the final stages of writing a GPL-licensed driver for the OLPC camera controller - and that he signed an NDA to obtain the requisite hardware documentation. As a result, he is, according to Theo de Raadt, "part of the problem."   
  
In the OLPC case, Theo's [criticism](<https://lwn.net/Articles/203570/>) has been centered upon (but not limited to) the driver for the Marvell wireless networking chip. Some very special things are being done with wireless on the OLPC, with the result that it will be able to function as a mesh network router with the CPU powered down. Enabling this involves a lot of close work with the chipset manufacturer - and a driver written under NDA. There are other NDA-covered drivers on the OLPC as well. 

~~Theo is unhappy that the OLPC will be, as he sees it, a closed system for OpenBSD.~~ [Mr. de Raadt has taken exception to the previous sentence, consider it removed]. But Theo is even more unhappy because, in his view, the OLPC project has squandered an opportunity to use its economic power with the manufacturers to force the hardware documentation out into the open. This failure is not just a lost opportunity; to Theo it also sends a message to other vendors that they need not worry about releasing hardware documentation. So, he says, the OLPC folks have not only failed to do the best they could; they have also actively made things worse for the free software community as a whole. 

The OLPC folks have several [responses](<http://www.gettysfamily.org/wordpress/?p=27>) to this criticism. The arrangement they have now, they say, is the best they could achieve within their particular set of goals - which, it should be remembered, is the provision of economical computers to children worldwide. OLPC was not founded with the primary goal of helping the free software community, though, in fact, that has been the result of much of its work. OLPC developers make the point that this computer will be one of the most open systems built in many years. The BIOS is free software, as is the VSA microcode which implements x86 emulation on the Geode CPU. The system's SD controller was redesigned (by Marvell) for the express purpose of allowing a driver to be written for it without having to sign the SD Association's particularly unpleasant NDAs. Even the firmware blob which runs on the wireless processor is slated for replacement with free software \- though that code does not exist at this point. 

Meanwhile, work continues on getting the hardware documentation released. It should be remembered, however, that much of this hardware does not actually exist yet. It would be rare indeed for a manufacturer to openly release this sort of information for a product which is not yet generally available. OLPC's plan appears to be to continue to work with the vendors to get the documentation released as the hardware comes onto the market. Heavy-handed pressure tactics, they feel, would be counterproductive in the end. 

The crux of the matter, thus, is this: if we accept that the community needs open hardware documentation to function as it should, what is the best way to get vendors to release that documentation? Some groups encourage ongoing engagement with these companies, with the intent of guiding them toward open source enlightenment. Under this line of thought, these companies will come to realize that the community will do great things with their hardware - growing the market - given the right information; they will see that it is in their economic interest to make the documentation available. 

The contrary argument is that this approach has never worked well, that hardware companies will never be brought around in this way. What is required, instead, is an intransigent insistence that the documentation must be released from the outset, and a refusal to sign NDAs to get it. Only when the vendors see themselves locked out of the free software market entirely will they realize that their interest lies in openness, not secrecy. Until that time, there is no reason to cooperate with uncooperative vendors; the preferred approach, instead, would appear to be to attempt to shame them publicly. 

There has been enough history of drivers written under NDA that it should be possible to come to some sort of conclusion as to which approach is more effective. The OpenBSD camp has arguably had some high-profile success with the public shame approach. Corporate conversions through quiet engagement tend to be more, say, quiet, however. Your editor would be most interested to hear about examples of companies changing their approach to hardware documentation as a result of working with free software developers under NDA. The question is not just academic: if we want to bring about an improvement in the hardware documentation situation, it behooves us to understand which tactics work best.
