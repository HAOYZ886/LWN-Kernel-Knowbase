---
title: "Copyleft-next and the kernel"
url: https://lwn.net/Articles/862611/
date: "July 13, 2021"
category: Copyright issues
author: "By Jake Edge July 13, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
July 13, 2021

The Linux kernel is, as a whole, licensed under the GPLv2, but various parts and pieces are licensed under other compatible licenses and/or dual-licensed. That picture was much murkier only a few years back, before the [SPDX in the kernel](<https://lwn.net/Articles/739183/>) project cleaned up the licensing information in most of the kernel source by specifying the licenses, by name rather than boilerplate text, directly in the files. A recent move to add yet another license into the mix is encountering some headwinds, but the license in question was already being used in a few kernel files, and has been for four years at this point. 

SPDX is more formally known as the [Software Package Data Exchange](<https://spdx.dev/>); it is a Linux Foundation project that has created an ""open standard for communicating software bill of material information, including provenance, license, security, and other related information"". In the kernel, SPDX identifiers are [used to identify the license](<https://www.kernel.org/doc/html/latest/process/license-rules.html>) as a comment at the top of a source file; for example: 

```
// SPDX-License-Identifier: GPL-2.0-only
```

For tooling reasons, SPDX headers in `.c` files use the "`//`" form of comments, while `.h` files use the more traditional "`/*` ... `*/`" form; both use license identifiers that refer to licenses that are stored in the [`LICENSES`](<https://elixir.bootlin.com/linux/v5.14-rc1/source/LICENSES>) directory of the kernel source tree. 

On July 7, Luis Chamberlain [posted](<https://lwn.net/ml/linux-kernel/20210707184310.3624761-1-mcgrof@kernel.org/>) a patch set that added the [copyleft-next 0.3.1](<https://github.com/copyleft-next/copyleft-next/blob/master/Releases/copyleft-next-0.3.1>) license to kernel tree and cleaned up four uses of that license in the tree. The [copyleft-next project](<https://github.com/copyleft-next/copyleft-next>) goes back a ways; it started in 2012 as a [GPLv3 fork](<https://lwn.net/Articles/505589/>) called GPL.next, but soon [took on a more neutral name](<https://lwn.net/Articles/537559/>). It is an attempt to create a strong copyleft license, in the mold of the GPL, but in simpler language that is easier to understand than GPLv3. It is explicitly written to be compatible with the GPL, so one could imagine kernel contributions that were solely licensed under copyleft-next. But, at least so far, all of the contributions using copyleft-next are dual-licensed as GPLv2 and higher (GPLv2+) as well. 

#### Backstory

Chamberlain's patch set did not come out of the blue. In an [earlier patch set](<https://lwn.net/ml/linux-kernel/20210702050543.2693141-1-mcgrof@kernel.org/>), he proposed a kernel self-test for sysfs. As with other tests he has written, this test was dual-licensed under GPLv2+ and copyleft-next (0.3.1). But Greg Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/YN6iSKCetBrk2y8V@kroah.com/>) that the GPLv2+ boilerplate in the 
```
(<https://lwn.net/ml/linux-kernel/20210702050543.2693141-2-mcgrof@kernel.org/>) was not needed, ""only the spdx line is needed"". He also asked that the copyleft-next license be removed: ""Please no, this is a totally different license :("". 

But, as Chamberlain [pointed out](<https://lwn.net/ml/linux-kernel/20210702190230.r46bck4vib7u3qo6@garbanzo/>), the use of copyleft-next in the kernel had been [discussed](<https://lore.kernel.org/lkml/1465929311-13509-1-git-send-email-mcgrof@kernel.org/>) back in 2016; Linus Torvalds had [no objection to its use](<https://lore.kernel.org/lkml/CA+55aFyhxcvD+q7tp+-yrSFDKfR0mOHgyEAe=f_94aKLsOu0Og@mail.gmail.com/>) and the comment text was [worked out](<https://lore.kernel.org/lkml/CAB=NE6VnftR-OR9iLFT3Lnbp4m35P0NFeGxNpRnAWvBbaemrFg@mail.gmail.com/>) with Alan Cox and Ted Ts'o at the time. In 2017, [test_sysctl was merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9308f2f9e7f055cf3934645ec622bb5259dc1c14>) using that text to indicate the dual license covering the code. During the discussion, Kroah-Hartman [acked a patch](<https://lore.kernel.org/lkml/20160701154258.GA32760@kroah.com/>) that added copyleft-next as an option in the kernel tree. 

The copyleft-next license is not listed in the kernel's `LICENSE` directory, however, so the SPDX lines in Chamberlain's test drivers only refer to GPLv2. That is not correct, as Kroah-Hartman [noted](<https://lwn.net/ml/linux-kernel/YN%2FrtmZbd6velB1L@kroah.com/>), but he also had a more fundamental objection: 

> And given that this is directly interacting with sysfs, which is GPLv2-only, trying to claim a different license on the code that tests it is going to be a total mess for any lawyer who wants to look into this. Just keep it simple please. 

Chamberlain, however, sees things differently with regard to the license compatibility: 

> The [fault] injection code I added follows the exact license for sysfs. The only interaction with the test_sysfs and sysfs is an exported symbol for a completion structure. The other dual gpl OR copyleft-next test drivers already present in the kernel also use exported symbols too, so I see nothing new here. 

#### Adding copyleft-next

There was a problem with copyleft-next not being in the kernel's license list, though, and thus the SPDX lines not truly reflecting the license status of the four files that had already been added (`lib/test_kmod.c`, `lib/test_sysctl.c`, and the corresponding shell scripts in `tools/testing/selftests`). For the C files, Chamberlain's [patch](<https://lwn.net/ml/linux-kernel/20210707184310.3624761-3-mcgrof@kernel.org/>) removes the boilerplate and updates the SPDX line as follows: 
[code]
        // SPDX-License-Identifier: GPL-2.0-or-later OR copyleft-next-0.3.1
```

The shell scripts have the equivalent change but, naturally, use "`#`" for the SPDX comment. 

Christoph Hellwig [replied](<https://lwn.net/ml/linux-kernel/20210708041446.GA17410@lst.de/>) to the cover letter of the patch set, asking about the need for a ""random weirdo license"" to be added to the kernel tree. Chamberlain [noted](<https://lwn.net/ml/linux-kernel/20210708190016.v6pm4d2vy2fzizfz@garbanzo/>) that the license is already being used on kernel code; in the [patch](<https://lwn.net/ml/linux-kernel/20210707184310.3624761-2-mcgrof@kernel.org/>) adding the license text, he also gave a list of a dozen copyleft-next benefits that he sees: 

>   * It is much shorter and simpler 
>   * It has an explicit patent license grant, unlike GPLv2 
>   * [...] 
>   * There is a built-in inbound=outbound policy for upstream contributions (cf. Apache License 2.0 section 5) 
>   * There are disincentives to engage in the controversial practice of copyleft/ proprietary dual-licensing 
>   * In 15 years copyleft expires, which can be advantageous for legacy code 
>   * There are explicit disincentives to bringing patent infringement claims accusing the licensed work of infringement (see 10b) 
>   * There is a cure period for licensees who are not compliant with the license (there is no cure opportunity in GPLv2) 
>   * copyleft-next has a 'built-in or-later' provision 

But Kroah-Hartman is [concerned](<https://lwn.net/ml/linux-kernel/YOaZohB%2F2Z3x5grc@kroah.com/>) about adding more licenses to the kernel; instead ""we should be trimming them down to be less as it makes things simpler and more obvious"". He noted that Chamberlain could switch the licenses of the four files, thus avoid needing to add copyleft-next. He also reiterated his arguments about the dual-licensing for test_sysfs, but said that he is sympathetic to proponents of copyleft-next: 

> [...] I do not want to see your "test_sysfs.c" module as a dual-licensed file, as that makes no sense whatsoever. It is directly testing GPL-v2-only code, so the attempt to dual license it makes no sense to me. How could anyone take that code and do anything with it under the copyleft-next license only? And where would that happen? 
> 
> I understand the appeal of copyleft-next in that it resolves many of the "grey" areas around gplv2, but given that no one is rushing to advise us to relicense all of the kernel with this thing, there is no need to encourage the spread of it given the added complexity and confusion that adding another license to our mix can only cause. 

The main organizer behind the copyleft-next project is Richard Fontana, but Bradley M. Kuhn worked on it as well, which he was quick to point out in a disclaimer on his [response](<https://lwn.net/ml/linux-kernel/YOcSwXkpzAFGucXM@ebb.org/>) to Kroah-Hartman. Kuhn noted that there is already a bunch of code in the kernel that is dual-licensed, many with either the two- or three-clause versions of the BSD license, which is evidently not a problem for kernel developers: ""There is no cogent argument that I can see that says '(GPLv2-only|{2,3}-Clause-BSD) is so special that it should be grandfathered in over other forms of dual licensing'."" Beyond that, though, since no one has done so, Kuhn wanted to ""be the first to advise"" the kernel community to switch the kernel license to copyleft-next, though he recognized the impossibility of that task. 

Tim Bird [pointed out](<https://lwn.net/ml/linux-kernel/BYAPR13MB2503F3D55FACFAE7868731BFFD199@BYAPR13MB2503.namprd13.prod.outlook.com/>) that the dual-licensing with BSD has resulted in ""the interchange of a lot of code between the BSD Unixes and Linux, that otherwise would not have happened"". It is very much in keeping with Torvalds's ""tit-for-tat compact"" to allow code improvements to flow both ways, he said. Kuhn [agreed](<https://lwn.net/ml/linux-kernel/YOc7dgBq%2FN5vDjhx@ebb.org/>) with Bird and hopes to see the same happen with projects that are released under copyleft-next, though there are far fewer of those. 

In the final analysis, as long as the other license is compatible with GPLv2, which copyleft-next is (so are BSD and others, of course), then it is up to the contributor to decide on the license(s), as Joe Perches [said](<https://lwn.net/ml/linux-kernel/d538c07ccb0176fe7c715bf4cb3b48d84b378c52.camel@perches.com/>). The situation is analogous to the [addition](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bc41a7f36469>) of the CC-BY-4.0 license to the kernel back in December; that was done because a documentation contributor wanted to dual-license their text. The contributor in this case, Chamberlain, [feels strongly](<https://lwn.net/ml/linux-kernel/20210708193334.uyndyk6mxpylv2qn@garbanzo/>) that copyleft-next is the right license for his code. He understands that there are other considerations for a large project like Linux, so he is taking a slow approach while trying to be conscious of the needs of others and the project as a whole. ""My personal development goal is I will embrace copyleft-next for anything new I write, and only use GPLv2 or another license when I am required to do so."" 

Of the benefits that he listed, the explicit patent grant is the most important to Chamberlain. He is concerned about a future without such a grant: 

> The license is one of the only few licenses (if not only?) which is GPLv2 compatible and also has an clear patent grant. I have reasons to believe, we as a community face serious challenges if we don't grow our collection of code with explicit patent grants. And so any new project I create will have such licenses. It is simply my preference, and if I can contribute code to Linux in a "safe place" to slowly build traction of it, then fantastic. 

Given that the license has been present in the kernel since 2017, and that it did not come in under cover of darkness, the changes Chamberlain has proposed seem like they should be relatively uncontroversial. There are certainly valid concerns about license proliferation, both within the kernel and without, but the main issue for the kernel community would seem to be satisfied by GPLv2 compatibility. It is possible that other compatible licenses will also need to be added to the `LICENSES` directory from time to time, but that seems a fairly small price to pay for useful contributions.
