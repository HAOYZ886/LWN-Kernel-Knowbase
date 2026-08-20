---
title: The push to save Itanium
url: https://lwn.net/Articles/950466/
date: "November 9, 2023"
category: "Architectures-ia64; Releases-6.7"
author: "By Jonathan Corbet November 9, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 9, 2023

It is (relatively) easy to add code to the kernel; it tends to be much harder to remove that code later. The most recent example of this dynamic can be seen in the story of the ia64 ("Itanium") architecture, support for which was removed during the 6.7 merge window. That removal has left a small group of dedicated ia64 users unhappy and clinging to a faint hope that this support could return in a year's time. 

At the end of the 1990s, it had become clear that 32-bit processors were approaching the end of their useful life for many applications; in particular, 32 bits is not enough to address the memory sizes that were beginning to show up on higher-end systems. In response, Intel launched into a program that it called "Merced" to create the successor to the x86. It was a RISC architecture, wholly incompatible with anything that Intel had sold before. But it was going to be the Next Big Thing because that was what Intel was doing. 

At the time, information about this new architecture was being held under nondisclosure agreements, and it was far from clear when Linux developers would be able to port the kernel to Merced, if ever. This was before Intel's [investment in Red Hat](</1998/1001/>) that signaled the beginning of the arrival of big money into Linux. It seemed entirely possible that Linux would be cut out of the processor that, we were all reliably informed, would be the future of computing; choices would be limited to Windows and proprietary Unix. 

That, of course, is not how things worked out. Intel became one of the earliest corporate supporters of Linux and ensured, under a project known as [Trillian](<https://www.redhat.com/en/about/press-releases/35>), that Linux ran well on this new architecture, which was eventually named ia64 (or "Itanium" in the sales literature). Initial Itanium support found its way into the 2.3.43 development kernel release in early 2000. The way was clear for our bright Linux-on-Itanium future. 

The only problem, of course, is that things didn't work out that way either. Early Itanium systems failed to perform at anything close to the speeds that the hype had promised. Meanwhile, AMD created the x86-64 architecture, adding 64-bit operation while maintaining as much compatibility with the vast world of deployed 32-bit software as possible. This new architecture quickly won over the market, forcing Intel to follow in AMD's footsteps; Itanium ended up as a nearly forgotten footnote. Some systems were sold, and Intel continued manufacturing the CPUs for many years, but their market was limited. Red Hat dropped ia64 support in 2010. 

Through all of this, the ia64 architecture code was maintained in the kernel, but interest dropped rapidly. In recent years, the ia64 code has often been seen as a drag on kernel development in general. After a bug originating in the ia64 code was [tracked down](<https://lwn.net/Articles/920259/>) in January, kernel developers started talking more seriously about just removing support for that architecture entirely. There was some discussion at the time, with a few hobbyist users complaining about the idea, but no changes were made then. 

The topic came back in May, though, When Ard Biesheuvel [pushed for the removal of ia64 support](<https://lore.kernel.org/all/CAMj1kXFCMh_578jniKpUtx_j8ByHnt=s7S+yQ+vGbKt9ud7+kQ@mail.gmail.com/>) from the kernel, saying that the architecture was impeding his work in the EFI subsystem: 

> As a maintainer, I feel uncomfortable asking contributors to build test their changes for Itanium, and boot testing is infeasible for most, even if some people are volunteering access to infrastructure for this purpose. In general, hacking on kernels or bootloaders (which is where the EFI pieces live) is tricky using remote access. 
> 
> The bottom line is that, while I know of at least 2 people (on cc) that test stuff on itanium, and package software for it, I don't think there are any actual users remaining, and so it is doubtful whether it is justified to ask people to spend time and effort on this. 

In that discussion, John Paul Adrian Glaubitz (who maintains the Debian ia64 port) [suggested](<https://lore.kernel.org/all/ff58a3e76e5102c94bb5946d99187b358def688a.camel@physik.fu-berlin.de/>) that ia64 support should be kept until after the next long-term-support kernel release, after which it could be dropped. That would, he said, maximize the amount of time in which ia64 would be supported for any remaining users out there. That is how it appears to have played out: during the 6.7 merge window, [ia64 support was removed](<https://git.kernel.org/linus/cf8e8658100d>). The ia64 story is now done, as far as Linux is concerned. 

Except that, seemingly, it is not. Shortly after ia64 support disappeared from the kernel, Frank Scheiner [complained](<https://lwn.net/ml/linux-kernel/8ff191a0-41fa-4f36-86e8-3d32ff3fe75c@web.de/>) to the mailing list, saying that he and others had been working to resolve the problems with this architecture and had been rewarded by seeing it removed anyway. Linus Torvalds [responded](<https://lwn.net/ml/linux-kernel/CAHk-=whFLZ67ffzt1juryCYcYz6eL_XjQF8WucDzwUR5H65+rA@mail.gmail.com/>) that he might be willing to see it come back — eventually: 

> So I'd be willing to come back to the "can we resurrect it" discussion, but not immediately - more along the lines of a "look, we've been maintaining it out of tree for a year, the other infrastructure is still alive, there is no impact on the rest of the kernel, can we please try again"? 

Scheiner was [not entirely pleased](<https://lwn.net/ml/linux-kernel/6e745433-d7eb-45df-b607-5589f1e04e86@web.de/>) with the removal of ia64 support, but Glaubitz [described](<https://lwn.net/ml/linux-kernel/396c3aebd6a382aa5a9cbf333f93d876467e12e9.camel@physik.fu-berlin.de/>) the one-year plan as ""very reasonable"". 

So the hobbyists who want to keep Linux support for this architecture alive, and who faced a difficult task before, have now seen the challenge become more severe. Maintaining support for an architecture out of tree is not a task for the faint of heart, especially as the mainline kernel goes forward with changes that had been held back by the need to keep ia64 working until now. To complicate the picture, as Tony Luck [pointed out](<https://lore.kernel.org/all/SJ1PR11MB60833AF98E6E9D3CD89AB393FC439@SJ1PR11MB6083.namprd11.prod.outlook.com/>) in May, it is entirely possible that future kernel changes may, when backported to the stable kernel updates, break ia64 in those kernels. Since nobody working on the stable updates is able to test ia64 systems (even if they wanted to), such problems could go unnoticed for some time. 

One should also not miss the other condition that Torvalds placed on a return of ia64: that ""the other infrastructure is still alive"". The ia64 enthusiasts did not miss that, so it is unsurprising that they were concerned when Adhemerval Zanella [proposed](<https://lwn.net/ml/libc-alpha/445c5c7c-315f-495d-81ce-a5bdc30e2b8e@linaro.org/>) removing ia64 support from the GNU C Library (glibc) — one of the most important pieces of other infrastructure. Zanella pointed out that the ia64 port is in poor shape, with a number of outstanding problems that seem unlikely to be solved. Scheiner [answered](<https://lwn.net/ml/libc-alpha/37fcf02a-6977-42ff-bed2-e1813f3f61ef@web.de/>) that it might be possible to provide (limited) access to ia64 machines for library testing, and asked for more time to address some of the problems. 

Zanella, though, followed up with [a patch](<https://lwn.net/ml/libc-alpha/20231104190225.2060378-1-adhemerval.zanella@linaro.org/>) to remove ia64 support. Scheiner [responded](<https://lwn.net/ml/libc-alpha/fbdeb854-ca42-4835-8cdc-fb130ea2b747@web.de/>): ""The speed this happens really surprises me and I hope there is no need to rush with this removal"". Other developers, though, including [Joseph Myers](<https://lwn.net/ml/libc-alpha/e5e9a6f8-f391-51f5-f4d9-384810c3c3aa@codesourcery.com/>), [Florian Weimer](<https://lwn.net/ml/libc-alpha/8734xiyhgu.fsf@mid.deneb.enyo.de/>), and glibc maintainer [Carlos O'Donell](<https://lwn.net/ml/libc-alpha/a53a3413-bb07-9d29-d692-777f54ad3065@redhat.com/>), are all in favor of dropping ia64 support. It would, thus, not be surprising to see the removal happen as soon as the 2.39 release, due in February, or at the latest in the release after that. 

That, needless to say, raises the bar for ia64 supporters even further. While one should never discount what a group of determined developers can accomplish, it is probably safe to conclude that ia64 support is gone from the kernel for good. Some may see this as a disappointment, but it is also a testament to how hard the community will work to keep an architecture alive even though it never had a lot of users and has almost none now. This support, arguably, could have been removed years ago without causing any great discomfort, but taking code out of the kernel is hard. As has been seen here, though, it is occasionally possible.
