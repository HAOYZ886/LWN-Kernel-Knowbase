---
title: Buying the kernel
url: https://lwn.net/Articles/106353/
date: "October 13, 2004"
category: Copyright issues
author: 
---

Long-time LWN readers have seen their share of, um, "interesting" Jeff Merkey quotes over the years. Mr. Merkey worked at Novell, but left to form the Timpanogas Research Group, which, at times, intended to sell "virtual network disk" technology, the Ute-Linux distribution, and a Netware-like kernel called MANOS. The company spent vast amounts of money in litigation with Novell, and was [ahead of the curve](<http://lwn.net/2000/1019/a/ute-linux.php3>) in the indemnification game: 

TRG grants indemnification against infringement claims by Novell to any commercial Linux companies, customers, distributors, etc. who use patches, NWFS, THOR, Ute-Linux, MANOS or any TRG technologies in their releases. 

Mr. Merkey claimed to have disposed of the Novell issue by means of having filed a sexual harassment suit against the company, but life was not to be so easy. The closure of Timpanogas was [announced](<http://lwn.net/2001/0704/a/nwfs.php3>) in 2001: 

I have dissolved TRG as a Utah Corporation and I am now focused on a variety of projects for various clients related to Linux development. Novell has recently threatened to try to take my house and assets if I post any more NWFS releases or MANOS. 

One would think that Mr. Merkey would have had enough intellectual property litigation for one life, but that appears to not be the case. He recently resurfaced on linux-kernel with [this interesting offer](<https://lwn.net/Articles/106358/>): 

We offer to kernel.org the sum of $50,000.00 US for a one time license to the Linux Kernel Source for a single snapshot of a single Linux version by release number. This offer must be accepted by **ALL** copyright holders and this snapshot will subsequently convert the GPL license into a BSD style license for the code. In other words, what we are asking for is the ability to snapshot kernel.org at 50K a pop for a license to each 2.<even number> release, then take any even number release private. 

The offer has spawned a number of side conversations on what an insultingly inadequate offer $50,000 really is. Certainly any number of companies would jump at the chance to pick up a non-GPL version of the kernel at that price. But such discussions - and the offer itself - miss the real point. 

Unlike many other large free software projects, the kernel does not require any sort of copyright assignment from contributors. Those who get code merged into the kernel retain their copyrights on that code. As a result, the kernel has hundreds - if not thousands - of copyright holders. Getting them all to agree on a licensing change would be a challenging task. Simply _finding_ them all is likely to be beyond just about anybody's capabilities. 

Critics of the kernel's organization claim that the lack of copyright assignment exposes the kernel to legal claims. They also state that the absence of a single copyright holder makes it difficult to enforce the GPL against those who fail to respect its terms. In response, one can point out that a copyright assignment would have been unlikely to deter the SCO Group from its campaign against IBM, and that the Netfilter team has been doing an admirable job of copyright enforcement. 

What widely distributed copyright ownership does do, however, is to make a relicensing of the code impractical, if not impossible. We need not worry that Linus will someday succumb to temptation and sell out the kernel. Some developers are suspicious of OSDL, but none fear that it will start selling off private versions of the kernel to well-heeled companies. For all that some people like to compare certain distributors with Microsoft, those distributors will never get into a position where they are shipping proprietary Linux kernels. 

Given this context, one wonders what Mr. Merkey thought he would be able to accomplish. There is no risk of him being able to buy himself a GPL exception for the kernel. The structure of the kernel's ownership is such that taking it private is not a practical possibility. This discussion is done; we must confess, however, to a certain curiosity about what Mr. Merkey's next scheme will be.
