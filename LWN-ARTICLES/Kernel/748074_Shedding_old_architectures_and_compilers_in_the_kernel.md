---
title: Shedding old architectures and compilers in the kernel
url: https://lwn.net/Articles/748074/
date: "February 26, 2018"
category: Architectures; Build system
author: "By Jonathan Corbet February 26, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 26, 2018

The kernel development process tends to be focused on addition: each new release supports more drivers, more features, and often new processor architectures. As a result, almost every kernel release has been larger than its predecessor. But occasionally even the kernel needs to slim down a bit. Upcoming kernel releases are likely to see the removal of support for a number of unloved architectures and, in an unrelated move, the removal of support for some older compilers. 

#### Architectures

The [Meta architecture](<https://en.wikipedia.org/wiki/Imagination_META>) was added to the 3.9 kernel as "metag" in 2013; it is a 32-bit architecture developed by Imagination Technologies. Unfortunately, at about the same time as the code was merged, Imagination Technologies bought MIPS Technologies and shifted its attention to the MIPS architecture. Since then, the kernel's support for Meta has languished, and it can only be built with the GCC 4.2.4 release, which is unsupported. On February 21, James Hogan, the developer who originally added the Meta port to the kernel, [proposed that it be removed](<https://lwn.net/Articles/747759/>), calling it ""essentially dead with no users"". 

The very next day, Arnd Bergmann, working entirely independently, [also proposed removing Meta](<https://lwn.net/Articles/748076/>). Bergmann, however, as is his way, took a rather wider view of things: he proposed that the removal of five architectures should be seriously considered. The other four were: 

  * The [Sunplus S+core](<https://en.wikipedia.org/wiki/S%2Bcore>) ("score") architecture entered the kernel for the 2.6.30 release in 2009. Since then, the maintainers for that architecture have moved on and no longer contribute changes to the kernel. GCC support for score was removed in the 5.0 release in 2015\. Bergman said: ""I conclude that this is dead in Linux and can be removed"". Nobody has spoken up in favor of retaining this architecture. 
  * The [Unicore 32](<https://en.wikipedia.org/wiki/Unicore>) architecture was merged for the 2.6.39 kernel in 2011. This architecture was a research project at Peking University. Bergmann noted that there was never a published GCC port and that the maintainer has not sent a pull request since 2014. This architecture, too, seems to lack users and nobody has spoken in favor of keeping it. 
  * Qualcomm's [Hexagon](<https://en.wikipedia.org/wiki/Qualcomm_Hexagon>) is a digital signal processor architecture; support for Hexagon entered the kernel in 2011 for the 3.2 release. Hexagon is a bit of a different case, in that the architecture is still in active use; Bergmann said that ""it is being actively used in all Snapdragon ARM SoCs, but the kernel code appears to be the result of a failed research project to make a standalone Hexagon SoC without an ARM core"". The GCC 4.5 port for this architecture never was merged. 

Richard Kuo [responded](<https://lwn.net/Articles/748077/>) in defense of the Hexagon architecture, saying: ""We still use the port internally for kicking the tools around and other research projects"". The GCC port is indeed abandoned, he said, but only because Qualcomm has moved to using LLVM to build both kernel and user-space code. Bergmann [responded](<https://lwn.net/Articles/748078/>) that, since there is still a maintainer who finds the code useful, it will remain in the kernel. He would like to put together a working LLVM toolchain to build this port, though. 
  * The [OpenRISC](<https://openrisc.io/>) architecture was merged in the 3.1 release, also in 2011. Bergmann observed that OpenRISC ""seems to have lost a bit of steam after RISC-V is rapidly taking over that niche, but there are chips out there and the design isn't going away"". He added it to his list because there is no upstream GCC support, but said that the OpenRISC GCC port is easy to find and the kernel code is being actively maintained. Philipp Wagner [responded](<https://lwn.net/Articles/748079/>) that the GCC code has not been upstreamed because of a missing copyright assignment from a significant developer; that code is in the process of being rewritten. The end result is that there is no danger of OpenRISC being removed from the kernel anytime soon. 

Bergmann also mentioned in passing that the FR-V and M32R architectures (both added prior to the beginning of the Git era) have been marked as being orphaned and should eventually be considered for removal. It quickly became apparent in the discussion, though, that nobody seems to care about those architectures. Finally, David Howells added support for the mn10300 architecture for 2.6.25 in 2008 and is still its official maintainer but, according to Bergmann, it ""doesn't seem any more active than the other two, the last real updates were in 2013"". Others in the discussion mentioned the tile (2.6.36 in 2010) and blackfin (2.6.21, 2007) as being unused at this point. 

The [plan](<https://lwn.net/Articles/748080/>) that emerged from this discussion is to remove score, unicore, metag, frv, and m32r in the 4.17 development cycle, while hexagon and openrisc will be retained. There will be a brief reprieve for blackfin and tile, which will be removed ""later this year"" unless a maintainer comes forward. And mn10300 will be marked for ""pending removal"" unless it gains support for recent chips. All told, there is likely to be quite a bit of code moving out of the kernel in the near future. 

#### Compilers

The [`changes.rst` file](<https://elixir.bootlin.com/linux/v4.15.6/source/Documentation/process/changes.rst>) in the kernel documentation currently states that the oldest supported version of GCC is 3.2, which was released in 2002. It has been some time, though, since anybody has actually succeeded in building a kernel with a compiler that old. In a discussion in early February, Bergmann [noted](<https://lwn.net/Articles/748082/>) that the oldest version known to work is 4.1 from 2006 — and only one determined developer is even known to have done that. The earliest practical compiler to build the kernel would appear to be 4.3 (2008), which is still supported in the SLES 11 distribution. 

Linus Torvalds, though, [said](<https://lwn.net/Articles/748083/>) that the real minimum version would need to be 4.5 (2010); that is the version that added the "`asm goto`" feature allowing inline assembly code to jump to labels in C code. Supporting compilers without this feature requires maintaining a fair amount of fallback code; `asm goto` is also increasingly needed for proper Meltdown/Spectre mitigation. Some developers would be happy to remove the fallback code, but there is a minor problem with that as [pointed out](<https://lwn.net/Articles/748084/>) by Kees Cook: LLVM doesn't support `asm goto`, and all of Android is built with LLVM. Somebody may need to add `asm goto` support to LLVM in the near future. 

Peter Zijlstra [would like](<https://lwn.net/Articles/748085/>) to go another step and require GCC 4.6, which added the `-fentry` feature; this replaces the old `mcount()` profiling hook with one that is called before any other function-entry code. That, too, would allow the removal of some old compatibility code. At that point, though, [according to Bergmann](<https://lwn.net/Articles/748086/>), it would make sense to make the minimum version be 4.8, since that is the version supported by a long list of long-term support distributions. But things might not even stop there, since the oldest version of GCC that is said to have support for the "retpoline" Spectre mitigation is 4.9, released in 2014. 

Nobody has yet made a decision on what the true minimum version of GCC needed to build the kernel will be so, for now, the documentation retains the fictional 3.2 number. That will certainly change someday. Meanwhile, anybody who is using older toolchains to build current kernels should probably be thinking about moving to something newer. 

(Thanks to Arnd Bergmann for answering a couple of questions for this article. It's also worth noting that he has recently updated [the extensive set of cross compilers](<https://www.kernel.org/pub/tools/crosstool/index.html>) available on kernel.org; older versions of those compilers can be had from [this page](<https://www.kernel.org/pub/tools/crosstool/files/bin/>).)
