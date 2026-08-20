---
title: The case of the supersized shebang
url: https://lwn.net/Articles/779997/
date: "February 18, 2019"
category: "Development model-Stable tree"
author: "By Jonathan Corbet February 18, 2019"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 18, 2019

Regressions are an unavoidable side effect of software development; the kernel is no different in that regard. The 5.0 kernel introduced a change in the handling of the "`#!`" (or "shebang") lines used to indicate which interpreter should handle an executable text file. The problem has been duly fixed, but the incident shows how easy it can be to introduce unexpected problems and highlights some areas where the kernel's development process does not work as well as we might like. 

By longstanding Unix convention, an attempt to execute a file that does not have a recognized binary format will result in that file being passed to an interpreter. By default, the interpreter is a shell, which will interpret the file as a shell script. If, however, the file starts with the characters "`#!`", the remainder of the first line will be treated as the name of the interpreter to use (and possibly arguments to be passed to that interpreter). This mechanism allows programs written in almost any interpreted language to be executed directly; the user need never know which interpreter is actually doing the work behind the scenes. 

> [**Update** : as noted in the comments, the above behavior is the result of both kernel and user-space code; in particular, the default to a shell is implemented within current shells and C libraries.] 

The array used to hold the shebang line is defined to be 128 bytes in length. That naturally leads to the question of what happens if the line exceeds that length. In current kernels, the line will simply be truncated to fit the buffer, after which execution proceeds as normal. Or, at least, as normal as can be expected given that part of the shebang line is now missing. Recently, Oleg Nesterov decided that this behavior is wrong; it could cause misinterpreted arguments or, should the truncated line happen to be the valid name of an interpreter executable in it own right, run the wrong interpreter entirely. He put together a patch ([merged for 5.0-rc1](<https://git.kernel.org/linus/8099b047ecc4>)) changing that behavior; the kernel would fail the attempt to find an alternative interpreter entirely in that situation, causing a fallback to the default shell. 

#### Trouble for NixOS

The [NixOS](<https://nixos.org/>) distribution, it seems, takes an unusual approach to the management of scripts. As noted in [a problem report](<https://lwn.net/ml/linux-kernel/CAN1fySViLgrLAHHfTHMef-Bkh73kUHKP-ava6TbgALeSE4LfFw@mail.gmail.com/>) posted by Samuel Dionne-Riel on February 13, NixOS scripts can have shebang lines like: 

```
#! /nix/store/mbwav8kz8b3y471wjsybgzw84mrh4js9-perl-5.28.1/bin/perl
           -I/nix/store/x6yyav38jgr924nkna62q3pkp0dgmzlx-perl5.28.1-File-Slurp-9999.25/lib/perl5/site_perl
           -I/nix/store/ha8v67sl8dac92r9z07vzr4gv1y9nwqz-perl5.28.1-Net-DBus-1.1.0/lib/perl5/site_perl
           -I/nix/store/dcrkvnjmwh69ljsvpbdjjdnqgwx90a9d-perl5.28.1-XML-Parser-2.44/lib/perl5/site_perl
           -I/nix/store/rmji88k2zz7h4zg97385bygcydrf2q8h-perl5.28.1-XML-Twig-3.52/lib/perl5/site_perl
```

This line has been split for (relative) ease of reading; it is all a single line in the files themselves. This line exceeds the maximum length by a fair amount, triggering the new code. The end result is that the Perl interpreter is not invoked as expected and the attempt to execute the file fails. User-space code reacts by passing the script to a shell, which rather messily fails to do the right thing with it. In other words, a change intended to prevent scripts from being passed to the wrong interpreter caused the system to start passing scripts to the wrong interpreter. The NixOS developers, rightly, saw this change as a regression; something that used to work no longer does with the 5.0 kernel. 

One might well wonder just how things worked before, since a truncated version of that shebang line is still wrong. It turns out that the Perl interpreter is able to detect this truncation; it rereads the first line itself and sets its arguments properly. As long as the interpreter itself is the correct one, things will work as expected. As of 5.0-rc1, though, the correct interpreter would no longer be invoked, and things went downhill from there. 

The kernel project's policy on this kind of change is clear, but Linus Torvalds [reiterated it](<https://lwn.net/ml/linux-kernel/CAHk-=wjtH5ombebghG-SDNGppJF4a5bZAUqndtZ3tauWjgw5_w@mail.gmail.com/>) in this case anyway: 

It doesn't matter if it "corrupted" things by truncating it. All that matters is "it used to work, now it doesn't" 

Yes, maybe it never *should* have worked. And yes, it's sad that people apparently had cases that depended on this odd behavior, but there we are. 

The change has since been [reverted](<https://git.kernel.org/linus/cb5b020a8d>), so NixOS will be able to run 5.0 kernels. There is work being done to achieve the original goal (preventing the kernel from possibly running the wrong interpreter) while not breaking existing users; that is proving harder than one might expect and will almost certainly have to wait for 5.1. 

#### Regressions in stable kernels

Had that been the end of the story, it would have been just another case of a regression introduced during the merge window, then corrected during the stabilization period. But, as it happens, this change found its way into the [4.20.8](<https://lwn.net/Articles/779597/>), [4.19.21](<https://lwn.net/Articles/779599/>), [4.14.99](<https://lwn.net/Articles/779600/>), and [4.9.156](<https://lwn.net/Articles/779601/>) stable kernel updates, despite the fact that neither the author nor the maintainer who merged it (Andrew Morton) had marked it for stable backporting. Morton [complained](<https://lwn.net/ml/linux-kernel/20190214122027.c0df36282d65dc9979248117@linux-foundation.org/>), noting that he had concluded that the patch should not be backported, but that backport had happened anyway. 

Not that long ago, the lack of an explicit tag would prevent a patch from being backported to the stable releases, but the situation has changed somewhat in recent years. Along with many of the other changes in that set of especially large stable kernel updates, Nesterov's patch had been automatically selected for backporting by [Sasha Levin's machine-learning system](<https://lwn.net/Articles/764647/>). Greg Kroah-Hartman [suggested](<https://lwn.net/ml/linux-kernel/20190215070022.GD14473@kroah.com/>) that concerned developers and users should have noticed this patch and complained before it was shipped: ""This came in through Sasha's tools, which give people a week or so to say 'hey, this isn't a stable patch!' and it seems everyone ignored that"". The implication is that, had people been paying attention, this regression would not have found its way into the stable updates. 

The patch in question was [flagged for backporting](<https://lwn.net/ml/linux-kernel/20190128154341.47195-301-sashal@kernel.org>) as part of a set of 304 selected for 4.20 on January 28\. It then found its way into the [4.20.8 review notification](<https://lwn.net/ml/linux-kernel/20190211141846.543045703@linuxfoundation.org>) on February 11\. That stable-release cycle gave developers and users a mere 352 patches to look over, but perhaps some understanding can be extended to those who didn't quite manage to evaluate the whole set in time. In truth, of course, there is little chance that anybody can truly look at that patch volume (multiplied by several major releases receiving stable updates at the same time) and pick out the bad patch. So some developers, [such as Michal Hocko](<https://lwn.net/ml/linux-kernel/20190215091000.GT4525@dhcp22.suse.cz/>), have said (again) that the process of moving patches into stable releases should be slower, perhaps waiting until those patches have appeared in a major release from Torvalds. That is especially true, [he said](<https://lwn.net/ml/linux-kernel/20190218125603.GO4525@dhcp22.suse.cz/>), of the ""nice-to-have"" patches that don't address problems users are complaining about. 

Levin [does not think that will help](<https://lwn.net/ml/linux-kernel/20190215151912.GA10616@sasha-vm/>): 

The fact is that many patches are not tested until they get to stable, whether we add them the same week they went upstream or months later. This is a great case for this: I doubt anyone but NixOS does this crazy thing with shebang lines, so who else would discover the bug? 

As a general rule, that might even be true, but it happens to not be in this case: the NixOS developers [discovered the problem](<https://github.com/NixOS/nixpkgs/issues/53672>) on January 8, and [filed a report](<https://bugzilla.kernel.org/show_bug.cgi?id=202497>) in the kernel bugzilla on February 2\. The commit causing the problem had been identified (through bisection) on February 3\. Shipping the regression in the stable updates had nothing to do with its discovery and reversion, in other words — the problem had already been identified well before the stable kernels shipped it. 

Even so, Levin [remains adamant](<https://lwn.net/ml/linux-kernel/20190215180026.GB10616@sasha-vm/>) that the process of automatically selecting patches for backporting is the right thing to do: 

The approach of manually deciding if a patch needs to go in stable is wrong and it doesn't scale. We need to beef up our testing story and make these decisions based off of that, and not our error-prone brains that introduced these bugs to begin with. 

This is undoubtedly an issue that will arise again; there are a great many fixes going into the kernel, and users of stable kernels (almost all of us) benefit from getting those fixes. But there are clearly some things that can be improved here. There was no test for this particular regression because it had never occurred to anybody that things could break in that way; we now know better, but no tests have been added yet. A kernel bugzilla instance that doesn't prevent a known-bad patch from getting into a stable release is clearly not doing its job; the kernel community as a whole lacks a convincing story on how bugs should be reported and tracked. The kernel development process works well in many ways, but that does not mean that it is without some glaring problems.
