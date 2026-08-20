---
title: "LinuxCon Japan: Making kernel developers less grumpy"
url: https://lwn.net/Articles/500443/
date: "June 6, 2012"
category: Development model
author: "By Jake Edge June 6, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 6, 2012

Greg Kroah-Hartman is on something of a mission: reducing the grumpiness factor among kernel developers, and maintainers in particular. His keynote at [LinuxCon Japan](<https://events.linuxfoundation.org/events/linuxcon-japan>) was meant to help the audience understand what the maintainers do, and how contributors' actions can sometimes result in grumpy maintainers. But, if contributors can follow the rules and make things easier on him, there are a number of things that he will promise to do on their behalf. 

[![\[Greg KH\]](https://static.lwn.net/images/conf/2012/lc-japan/lcj-gregkh-sm.jpg)](<https://lwn.net/Articles/500550/>) He called the Linux kernel the ""largest software development project ever"" and noted that its development pace is ""unprecedented"". From 3.0 to 3.4, some 2833 developers from at least 373 companies contributed. In that year (from May 2011 to May 2012), the kernel had a change rate of 5.79 changes per hour. But the rate keeps increasing and if you look at just the 3.4 cycle, the rate is 7.21 changes per hour. That is, of course, just patches that are accepted into the mainline, so it doesn't count those patches that are rejected. 

Developers typically send their changes to the maintainer of the file that is being changed. Those maintainers, who number around 700, feed those changes up to the 130 subsystem maintainers. From there, the patches make their way into linux-next, then to Linus Torvalds, and, eventually the mainline--_if_ they get accepted at each step along the way. 

So, in order to see why some patches might not get accepted, he looked at those that he received in the last two weeks, which coincided with the 3.5 merge window. The merge window is a time when he really shouldn't be getting many patches. He should have received them all earlier in the cycle so that he could potentially pass them on to Torvalds during the merge window. But, he said, he got 487 patches in that two-week period, many with a wide variety of problems, and some of those from core kernel developers who should know better. 

#### Broken patches

With that, he launched into a description of some of the broken patches he got. One patch was labeled "patch 48/48" (i.e. the last patch in a set of 48) but all of the other pieces were missing. He also got a patch series with no order specified, which means that he would have to guess at the order and undoubtedly get it wrong. The alternative is to ignore the patch entirely. He also got a ten-patch set that was missing patch two in the series. 

Another patch came in an email with a signature claiming that it was confidential. He actually sees that one a lot, he said, and there is nothing he can do with those kinds of patches. Linux development is done in the open and you can't send a confidential email to mailing lists or get a confidential patch merged. Obviously, it is boilerplate that gets added somewhere in the email process, but it has to be removed before the patch can be used. 

There are also malformed patches that end up in his inbox, including those with tabs converted to spaces. Microsoft Exchange does that, he said, so if that's a problem in your environment, do what IBM, Microsoft, and others do: put a Linux box in the corner for the developers to use to send their mail. Sometimes the leading spaces have been stripped off the diff or the diff is not in unified format. Linux developers have gotten good at raw editing diff format, he said, which is scary in itself, but they shouldn't have to do that. 

Patches are also created in the wrong directory, like down in a driver directory for example. He got a patch created in `/usr/src/linux-2.6.32` and noted that there were multiple things wrong with that, including the age of the source tree and that it implied it was being built by root. The latter is very dangerous as there was a bug in the Linux build process at one point that would delete the entire root filesystem if it was run as root. None of the core developers noticed because they don't build as root. Suggestions that the bug be left in as a deterrent were ignored, but things like that can happen. 

In addition, patches came in that were made against a different tree than any he would expect. He got a patch made against the SCSI development tree, for reasons unknown because it had nothing to do with SCSI. 

Then there are those that don't have the right coding style. In one case, the coding style was wrong and the developer acknowledged that but wanted him to take the patch anyway. That gives the impression of ""we don't care, take our code anyway"", he said. There are tools to help find and fix those kinds of problems, so there is no excuse: ""send it in the right coding style"". 

Something he sees much more than he should are patches that don't even compile. The submitter clearly hasn't even built the patch, he said. Or there are patch sets that break the build in 3/6 but then fix it in 6/6. He even got a patch that broke the build in 5/8 but contained a note that sometime in the future the submitter would send changes to fix it. Another patch had obviously wrong kernel-doc in it that would cause failures building the documentation, so it was clear that the contributor had never even tried to run the kernel-doc extraction tool. 

One of the patches he got ""had nothing to do with me"". It was an x86 core kernel patch, which is not an area of the kernel he has ever dealt with. But the patch was sent only to him. ""I get odd patches"" a lot, he said. 

The last patch he mentioned was 450K in size, with 4500 lines added. Somebody suggested that it be broken up, but in the meantime several maintainers actually reviewed it, so the submitter didn't really learn from that mistake. 

All of this occurred during a ""calm two weeks"", he said. These are examples of what maintainers deal with on a weekly basis and explains why they can be grumpy. That said, he did note that this is the ""best job I've ever had"", but that's not to say it couldn't be improved. 

If someone sends him a patch and he accepts it, that means he may have to maintain it and fix bugs in it down the road. So it's in his self interest to ignore the patch, which is an interesting dynamic, he said. The way around that is to ""give me no excuse to reject your patch""; it is as simple as that, really. 

#### Rules

Kroah-Hartman then laid out the rules that contributors need to follow in order to avoid the kinds of problems he described. Use `checkpatch.pl`, he said, because he will run it on your patch and it is a waste of his time to have to forward the results back when it fails. Send the patch to the right people and there is even a script available (`get_maintainer.pl`) to list the proper people and mailing lists where a patch should be sent. 

Send the patch with a proper subject that is ""short, sweet, and descriptive"" because it is going to be in the kernel changelog. It should not be something like ""fix bugs in driver 1/10"". In addition, the changelog comment should clearly say what the patch does, but also _why_ it is needed. 

Make small changes in patches. You don't replace the scheduler in one patch, he said, you do it over five years. Small patches make it easier for reviewers and easier for maintainers to accept. In a ten-patch series, he might accept the first three, which means that the submitter just needs to continue working on the last seven. The best thing to do is to make the patch ""obviously correct"", which makes it easy for a maintainer to accept it. 

Echoing the problems he listed earlier, he said that patches should say what tree they are based on. In addition, the order of the patches is important, as is not breaking the build. The latter ""seems like it would be obvious"" but he has seen too many patches that fail that test. To the extent that you can, make sure that the patch works. It is fine to submit patches for hardware that you don't have access to, but you should test on any hardware that you do have. 

Review comments should not be ignored, he said. It is simply common courtesy if he takes time to review the code that those comments should be acted upon or responded to. It's fine to disagree with review comments, but submitters need to say why they disagree. If a patch gets resent, it should be accompanied with a reason for doing so. When reviewer's comments are ignored, they are unlikely to review code the next time. 

#### Maintainer's role

When you follow those rules there are certain things you can expect from him, Kroah-Hartman said, and that you should expect from the other maintainers as well. That statement may make other maintainers mad, he joked, but it is reasonable to expect certain things. For his part, he will review patches within one or two weeks. Other maintainers do an even better job than that, he said, specifically pointing to David Miller as one who often reviews code within 48 hours of its submission. If you don't get a response to a patch within a week, it is fine to ask him what the status is. 

He can't promise that he will always give constructive criticism, but he will always give ""semi-constructive criticism"". Sometimes he is tired or grumpy, so he can't quite get to the full "constructive" level. He will also keep submitters informed of the status of their patch. He has scripts that will help him do so, and let the submitter know when the patch gets merged into his tree or accepted into the mainline. That is unlike some other maintainers, he said, where he has submitted patches that just drop into a ""big black hole"" before eventually popping up in the mainline three months later. 

He ended by putting up a [quote from Torvalds](<https://lwn.net/Articles/490383/>) (""Publicly making fun of people is half the fun of open source programming. ..."") that was made as a comment on one of Kroah-Hartman's Google+ postings. The [post](<https://plus.google.com/111049168280159033135/posts/5YtkxtuRXTy>) was a rant about a driver that had been submitted, which even contained comments suggesting that it should not be submitted upstream. He felt bad about publicly posting that at first, but Torvalds's comment made him rethink that. 

Because kernel development is done in the open, we are taking ""personal pride in the work we do"". As the code comment indicated, the driver developer didn't think it should be submitted because they realized the code was not in the proper shape to do so. It is that pride in the work that ""makes Linux the best engineering project ever"", he said. Sometimes public mocking is part of the process and can actually help instill that pride more widely. 

[ The author would like to thank the Linux Foundation for assistance with his travel to Yokohama. ]
