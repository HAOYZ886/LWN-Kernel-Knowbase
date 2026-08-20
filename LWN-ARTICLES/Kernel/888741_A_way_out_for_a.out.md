---
title: A way out for a.out
url: https://lwn.net/Articles/888741/
date: "March 24, 2022"
category: a.out
author: "By Jonathan Corbet March 24, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 24, 2022

The [a.out executable format](<https://en.wikipedia.org/wiki/A.out>) dates back to the earliest days of Linux — and before. It has not been used in any serious way for decades, but support still exists in the Linux kernel and has resisted all attempts at its removal. Back in January, Borislav Petkov [tried yet again](<https://lwn.net/ml/linux-kernel/20220113160115.5375-1-bp@alien8.de/>) to delete support for this format, leading to another extended discussion. There is one difference this time around, though: the effort to get rid of a.out support might just succeed. 

The a.out format dates back to the first edition of Unix. When MINIX came along, it naturally used that format for its executable files; that, in turn, led to a.out being used in Linux as well. It is a simple format, and its implementation on Linux was even simpler; among other things, every Linux shared library had to be centrally assigned its own portion of the address space, since libraries could not be relocated at run time. Still, Linux used a.out for some time, until support for the newfangled ELF format was first added to the 0.99.13 development kernel in 1993. 

There was a time when the crazier people among us manually converted our Slackware systems from a.out to ELF in order to be able to try it out and gain the benefits before distributions were updated. They still bear the scars from that time. Not that your editor would ever admit to knowing anybody who would have engaged in any such activity. 

ELF has been the standard executable format for Linux on most architectures since 1995. One might think that would have provided enough time for any users of a.out binaries to grudgingly move on to ELF; its adoption can probably be judged to not be a passing fad at this point. But, in the real world, surprises lurk. 

The initial conversation on removing a.out faded out fairly quickly, but was restarted when Eric Biederman [posted a patch](<https://lwn.net/ml/linux-kernel/87ilsmdhb5.fsf_-_@email.froward.int.ebiederm.org/>) disabling the building of a.out support on the two architectures (Alpha and m68k) that still enable it by default. This patch did not actually remove the support, it just turned it off to see if anybody would scream. If protests were raised, support could be turned back on quickly and easily; otherwise a full removal could be done. 

Linus Torvalds quickly [responded](<https://lwn.net/ml/linux-kernel/CAHk-=wg+TYsns5JvNds6BVG7ezdg8uM_z9m8uJBcRDANdd7csw@mail.gmail.com/>) that he was ""pretty sure we can't do this"". He pointed out that the native executable format on Alpha systems running Digital's Unix was essentially a.out, even if it went by the newer name of ECOFF. The Linux a.out loader can run ECOFF programs by virtue of ignoring some of the newer ECOFF features; taking it out would break any systems still using that support. One might think that the number of users who are still using Alpha CPUs, running ECOFF binaries on those CPUs, and tracking current kernel releases might be a fairly small number, but one never knows. 

Kees Cook did [do a little research](<https://lwn.net/ml/linux-kernel/202203091358.775E158C@keescook/>) on, seemingly, the only distribution that still supports Alpha (Gentoo) and found that the only ECOFF files present contained firmware, which does not run on the CPU anyway. He concluded that there would be no harm from removing a.out support on this platform: ""Let's do it"". 

It seems to be a universal rule that somebody always has to come along to ruin the party. In this case, just as it seemed like there were no further obstacles to the removal of a.out, James Jones [showed up](<https://lwn.net/ml/linux-kernel/4c449fab-8135-5057-7d2c-7b948ce130cc@theinnocuous.com/>) to let it be known that he was still using a.out: 

> The use case is running an old set of tools to build programs for the Atari Jaguar. Namely, Atari's assembler (mac) and linker (aln). The alternative is running windows versions in dosbox, or using some replacements that have been developed based on an even older, less-featureful version of the source code for mac and aln, but which still haven't managed to add back in all the features needed to build some programs or use the Atari debugging tools (Also available in a.out only). 

He gave some more details on why he still needed to use these tools in [this followup message](<https://lwn.net/ml/linux-kernel/785a6348-e1c5-7900-78bc-af8b6923cc8a@theinnocuous.com/>)

Moving a program to a new executable format is normally just a matter of rebuilding the program. But rebuilding is a rather more difficult task in the absence of the source code. Finding oneself stuck trying to run a decades-old binary on modern systems is just one of the many joys reserved for users of proprietary software — but many people have found themselves in that position at one time or another. They are legitimate Linux users, and there is no desire to break their systems. So Petkov duly [gave up](<https://lwn.net/ml/linux-kernel/Yi3b%2FGpUuhkI5lv7@zn.tnic/>) and asked only that Jones document his usage of a.out for the benefit of those who might try to remove it in the future. 

Cook, however, was not so quick to throw in the towel; he looked at the programs in question and [concluded](<https://lwn.net/ml/linux-kernel/202203151150.1CDB1D8DA@keescook/>) that it might be possible to write an ELF wrapper program that could load and run old a.out binaries. One day later, he [posted such a program](<https://lwn.net/ml/linux-kernel/202203161523.857B469@keescook/>), noting that it could run the `aln` program at least far enough for it to complain about its command-line arguments. Jones [tried it out](<https://lwn.net/ml/linux-kernel/59fb3d14-423a-d39b-04cb-1e60ee67d3f1@theinnocuous.com/>) and was pleased with the results: 

> Yes, works perfectly, thank you. I like the idea of using this much better than getting emailed every time someone wants to delete the a.out code again. Consider my use case withdrawn. I've already pushed an update to my jaguar-sdk project to use this tool instead. 

So removal of a.out support is [back in the plans](<https://lwn.net/ml/linux-kernel/YjMKsM6LzEmpGTzN@zn.tnic/>) for the 5.18 kernel. Perhaps the effort will succeed this time, though there are still no guarantees; there may be a.out users who are unaware of the coming apocalypse and haven't yet made their objections known. If those users are unable or unwilling to use Cook's wrapper utility, the excision of a.out support may, once again, have to be deferred to some future kernel release.
