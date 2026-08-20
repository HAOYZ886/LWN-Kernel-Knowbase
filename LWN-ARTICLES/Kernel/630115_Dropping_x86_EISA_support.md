---
title: Dropping x86 EISA support
url: https://lwn.net/Articles/630115/
date: "January 21, 2015"
category: EISA
author: "By Jake Edge January 21, 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
January 21, 2015

It is clear that Paul Gortmaker thought there was a pretty good chance to get rid of some old, unloved code when he [proposed](<https://lwn.net/Articles/630172/>) dropping support for the [EISA bus](<http://en.wikipedia.org/wiki/Extended_Industry_Standard_Architecture>) from 32-bit x86 kernels. As he noted, when support for the [MCA bus](<http://en.wikipedia.org/wiki/Micro_Channel_architecture>) was dropped in 2012, Linus Torvalds [mused](<https://lkml.org/lkml/2012/5/17/278>) that perhaps EISA could follow suit ""some day"". Obviously Gortmaker hoped that day had come, at least for the x86 architecture, but it seems he was a bit premature. 

As Gortmaker pointed out, there are some architectures that are essentially ""frozen in time (from a hardware perspective)""—he mentioned Alpha and PA-RISC as examples—so EISA support cannot be completely removed from the tree (as MCA was). Removing it from x86 did not save much in the way of code—it only deleted a little over 100 lines—but he had something else in mind: 

Given that it is 20 years on since its demise, and the above specs might seem just barely acceptable for a wireless router today, lets stop forcing everyone to build EISA infrastructure and assoc. drivers during their routine build coverage testing for no value whatsoever. 

But Maciej W. Rozycki was [not on board](<https://lwn.net/Articles/630128/>) with the removal, noting that it is needed ""to support EISA FDDI [[Fiber Distributed Data Interface](<http://en.wikipedia.org/wiki/FDDI>)] equipment I maintain if nothing else"". He suggested that perhaps it could be hidden behind a configuration option for ""more exotic stuff"" so that not everyone needed to build and test it. 

Unsurprisingly, Torvalds was quick to [put the kibosh](<https://lwn.net/Articles/630129/>) on EISA's removal: ""So if we actually have a user, and it works, then no, we're not removing EISA support"". But it is instructive to consider what might have happened if Rozycki had not posted his disagreement. It seems quite possible that if no one spoke up for EISA on x86, it might well have been removed. 

There is always some tension in the kernel community between those who want to clean up and clear out "legacy" code and those who want to see it continue to live in the mainline tree. There is a cost associated with maintaining legacy code, though, even if it rarely needs to change, and it does continue to get built as part of various kernel-wide testing efforts. That puts some (possibly small) amount of burden on many other kernel developers, most of whom are not interested in the old code at all. 

As certain kinds of hardware start to disappear entirely—from the kernel developers' consciousness, at a minimum—it behooves those using Linux on that hardware to pay attention to the kernel mailing list. As seen here, real users who do speak up will likely be able to block efforts to remove support, but timely responses will be needed. If a kernel release cycle or two goes by, it may well be too late.
