---
title: Kernel code on the chopping block
url: https://lwn.net/Articles/920259/
date: "January 19, 2023"
category: "Architectures-SuperH; Architectures-ia64; Filesystems-JFS"
author: "By Jonathan Corbet January 19, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 19, 2023

Code that is added to the kernel can stay there for a long time; there is code in current kernels that has been present for over 30 years. Nothing is forever, though. The kernel development community is currently discussing the removal of two architectures and one filesystem, all of which seem to have mostly fallen out of use. But, as we will see, removal of code from the kernel is not easy and is subject to reconsideration even after it happens. 

#### Not-so-SuperH

Hitachi's SuperH architecture dates back to the 1990s; it is a 32-bit processor that was intended for embedded applications. Support for SuperH was first added (in minimal form) to the kernel in the 2.3.16 release in September 1999. While it was popular for some years after its introduction, SuperH was eventually eclipsed by Arm. SuperH processors are still [produced by Renesas](<https://www.renesas.com/us/en/products/microcontrollers-microprocessors/other-mcus-mpus/superh-risc-engine-family-mcus>), but they are not recommended for use in new designs. 

Around 2015 there was [an attempt](<https://lwn.net/Articles/647636/>) to reinvigorate this architecture, inspired by the fact that the patents covering its design were expiring. The result was [a design](<https://j-core.org/>) for a processor called the J-Core J2, which was intended to be built as entirely open hardware. J2 did not really take off, though, perhaps because interest in 32-bit processors was fading quickly in those times; the [J-Core news page](<https://j-core.org/news.html>) was last updated in 2016. [This message](<https://lwn.net/ml/linux-kernel/7329212f-b1a0-41eb-99b3-a56eb1d23138@landley.net/>) from Rob Landley suggests that this project is not entirely dead. Still, despite the appeal of an entirely open processor, it does not appear that J-Core will be the salvation of the SuperH architecture. 

More recently, Christoph Hellwig posted [a patch series](<https://lwn.net/ml/linux-kernel/20230113062339.1909087-1-hch@lst.de/>) removing support for the SuperH architecture from the kernel, saying: 

> arch/sh has been a long drag because it supports a lot of SOCs, and most of them haven't even been converted to device tree infrastructure. These SOCs are generally obsolete as well, and all of the support has been barely maintained for almost 10 years, and not at all for more than 1 year. 

John Paul Adrian Glaubitz [objected](<https://lwn.net/ml/linux-kernel/11e2e0a8-eabe-2d8c-d612-9cdd4bcc3648@physik.fu-berlin.de/>) to this removal, noting that he is still maintaining the Debian SuperH port. ""It's a bit disappointing that people keep hammering on it. It works fine for me."" Geert Uytterhoeven [added](<https://lwn.net/ml/linux-kernel/CAMuHMdUcnP6a9Ch5=_CMPq-io-YWK5pshkOT2nZmP1hvNcwBAg@mail.gmail.com/>) that the problem with SuperH isn't really a lack of users or even developers creating patches, it's the fact that there is no maintainer to apply those patches. 

A likely outcome to this discussion is that Glaubitz, along with Landley, will try to take over maintenance of the kernel's SuperH port. They will have their work cut out for them; among other things, SuperH needs a proper conversion to use device trees and the removal of support for many obsolete CPUs and boards. But, if the new maintainers can get SuperH back on track, its days in the kernel may not be numbered quite yet. 

#### The last sinking of the Itanic

In the late 1990s, there was a lot of talk of Intel's new processor architecture, which was known by the codename "Merced" at the time. This CPU was going to be the future of computing, it would be so much better that it would quickly dethrone x86. But details about Merced were scarce, and it was not at all clear that there would be enough information available to create a Linux port for this architecture. There were widespread worries that, by denying access to information on Merced, Intel and Microsoft would sideline Linux just as it was beginning to gain some real commercial momentum. 

Needless to say, that is not what happened. Intel, instead, [became an early investor in Red Hat](</1998/1001/>) in 1998. When Merced, later branded "Itanium", hit the market, Linux had first-class support for it. The Itanium (ia64) architecture first showed up in the 2.3.43 development kernel release in early 2000. Working Itanium processors took somewhat longer and, by the time they were widely available, they had long since been surpassed by x86-64, initially developed by AMD. Itanium was never a successful product line and no Itanium CPUs are manufactured now. 

Itanium support remains in the kernel, though, but it's not clear how much longer that will last. The most recent discussion about its future happened as a result of [this patch](<https://lwn.net/ml/linux-fsdevel/CAGudoHHx0Nqg6DE70zAVA75eV-HXfWyhVMWZ-aSeOofkA_=WdA@mail.gmail.com/>) from Mateusz Guzik, who had tracked down a locking performance problem that, seemingly, originated in a `cpu_relax()` call that is only needed by the Itanium architecture. That led Tony Luck, once a maintainer of that architecture in the kernel, to [ask](<https://lwn.net/ml/linux-fsdevel/SJ1PR11MB6083368BCA43E5B0D2822FD3FCC29@SJ1PR11MB6083.namprd11.prod.outlook.com/>): 

> Is it time yet for: 
> 
> $ git rm -r arch/ia64 

Ard Biesheuvel [concurred](<https://lwn.net/ml/linux-fsdevel/CAMj1kXEqbMEcrKYzz2-huLPMnotPoxFY8adyH=Xb4Ex8o98x-w@mail.gmail.com/>), noting that he is unable to do any sort of testing to ensure that EFI changes work on the Itanium architecture. On the other hand, [Jessica Clarke](<https://lwn.net/ml/linux-fsdevel/Y8HDzzDaP5uY0v8K@Jessicas-MacBook-Pro/>) and [Glaubitz](<https://lwn.net/ml/linux-fsdevel/db6937a1-e817-2d7b-0062-9aff012bb3e8@physik.fu-berlin.de/>) both pointed out that Debian has a working Itanium port; Glaubitz added that Gentoo also supports the architecture. Luck [suggested](<https://lwn.net/ml/linux-fsdevel/SJ1PR11MB60832EF4EA3D528533100F8DFCC29@SJ1PR11MB6083.namprd11.prod.outlook.com/>) that keeping these systems going was a wasted effort: 

> Are there people actually running production systems on ia64 that also update to v6.x kernels? 
> 
> If so, why? Just scrap the machine and replace with almost anything else. You'll cover the cost of the new machine in short order with the savings on your power bill. 

The [answer](<https://lwn.net/ml/linux-fsdevel/53F2E30D-FE1D-4710-B5FC-049905A7158E@jrtc27.com/>) was "hobbyists", who like keeping such systems running. Biesheuvel [sympathized](<https://lwn.net/ml/linux-fsdevel/CAMj1kXH1SjPrPWyQbsYUHhVfgWH_p-sf-mhbeKMQn-QyOjBRng@mail.gmail.com/>) with the hobbyists, but asked: 

> However, the question is not how you or I choose to spend (or waste) their time. The question is whether it is reasonable *as a community* to insist that everyone who contributes a cross-architecture change also has to ensure that obsolete architectures such as ia64 or alpha are not left behind. 

He pointed to the original topic of the thread as an example: locking was being slowed as the result of an Itanium-specific problem, so developers need to find a way to speed up the locking that doesn't introduce regressions on a processor that nobody has access to. Or perhaps not: Luck [suggested](<https://lwn.net/ml/linux-fsdevel/SJ1PR11MB6083B48A2B2114EF833D69E2FCC29@SJ1PR11MB6083.namprd11.prod.outlook.com/>) how such a fix could be structured, but Linus Torvalds [answered](<https://lwn.net/ml/linux-fsdevel/CAHk-=wgTsc5z3cPo7+t2kRO1uRQML1w_o72nefyHOh8VMhqu0A@mail.gmail.com/>) that it simply wasn't worth the effort. 

Nobody yet has yet said that Itanium support can actually be removed, so it is not clear when that will happen. Sooner seems more likely than later, though. As developers stop even their minimal support for this architecture, it will fade into an increasingly unmaintainable state. Support for this unloved architecture has been in the kernel for 23 years; it's not likely to stay for many more. 

#### JFS

Once upon a time, journaling was the hot new feature for filesystems, and Linux didn't have any filesystems that supported it; system administrators had to be far more familiar with the `fsck` tool than is normally required now. That hole was first filled by ReiserFS, which was merged for the 2.4.0.4 in early February 2002\. But it was quickly followed by JFS, which entered the kernel less than three weeks later. JFS came out of IBM and was the main filesystem for the AIX operating system; it, too, supported journaling. 

JFS never become one of the top Linux filesystems, though it did pop up here and there; the [MythTV](<https://www.mythtv.org/>) project still suggests it in various places in its documentation. But JFS has not seen any real development for years and it's not clear that there are many users left. Hellwig, pointing out that JFS will slow down the [folio conversion](<https://lwn.net/Articles/893512/>) effort, [asked](<https://lwn.net/ml/linux-kernel/Y8DvK281ii6yPRcW@infradead.org/>) whether it should just be removed. 

This filesystem seems to have few defenders. Dave Kleikamp, who originally added it in 2002 and who is still listed as its maintainer, [responded](<https://lwn.net/ml/linux-kernel/f99e5221-4493-dba3-3e80-e85ada6b3545@oracle.com/>): ""Obviously, I haven't put much effort into JFS in a long time and I would not miss it if it were to be removed"". No patches doing the removal have been posted but, in the absence of an impassioned defense of JFS from somewhere, one can expect it to, at a minimum, be marked as orphaned in the near future. 

#### The zombie pktcdvd driver

[Packet writing](<https://en.wikipedia.org/wiki/Packet_writing>) is a protocol that can be used for the writing of recordable optical media (younger readers should ask their parents about optical media; it used to be important). The kernel gained support for packet writing in late 2004. As of 2016, though, the pktcdvd driver was unmaintained and unloved; the regular block drivers had long since gained the ability to drive all optical devices, and the packet-writing driver, it seemed, was no longer needed. It was [marked deprecated](<http://git.kernel.org/linus/5a8b187c61e9>) in the 4.10 release with a promise that it would be removed ""after a release or two"". 

A full 33 releases later, this driver was [actually removed](<https://git.kernel.org/linus/f40eb99897af>) during the 6.2 merge window. That, however, led to [a complaint](<https://lwn.net/ml/linux-kernel/20230104190115.ceglfefco475ev6c@pali/>) from Pali Rohár, who said: 

> This 6.2-rc1 release is breaking support for CD-RW, DVD-RW, DVD+RW and partially also DVD-RAM medias because of pktcdvd driver removal from the kernel in this release. Driver is still used and userspace tools for it are part of the udftools project, which is still under active maintenance. More people already informed me about this "surprise". 

Both [Torvalds](<https://lwn.net/ml/linux-kernel/CAHk-=wh1x-gbmE72xBPcEnchvmPn=13mU--+7Cfay0dNHCxxuw@mail.gmail.com/>) and block subsystem maintainer [Jens Axboe](<https://lwn.net/ml/linux-kernel/90eb90da-2679-cac0-979d-6ba0cc8ccbb8@kernel.dk/>) questioned the need for this driver, but it became clear that it still provides some functionality (specifically ensuring that all writes are exactly 32KB in size) that projects like udftools need. So [pktcdvd came back](<https://git.kernel.org/linus/4b83e99ee709>). There was some talk of putting the necessary logic into the SCSI CD driver, but, as Torvalds [noted](<https://lwn.net/ml/linux-kernel/CAHk-=wjJ=wD5D80hkWNCjJqS+djckAL+nXhXaHaiFzMAMve3rA@mail.gmail.com/>), ""nobody is going to be motivated to do any development in this area, and the best we can do is probably to just keep it limping along"". If, at some point, this driver fails to limp properly, developers may start considering its removal again.
