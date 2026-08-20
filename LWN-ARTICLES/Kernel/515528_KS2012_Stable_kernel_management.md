---
title: "KS2012: Stable kernel management"
url: https://lwn.net/Articles/515528/
date: "September 12, 2012"
category: "Development model-Stable tree"
author: "By Michael Kerrisk September 12, 2012 2012 Kernel Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Michael Kerrisk**  
September 12, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

In a short session toward the close of day one of the 2012 Kernel Summit, Greg Kroah-Hartman, the maintainer of the stable kernel series, relayed one of his concerns about the stable kernel and sought questions and feedback from those present. 

Greg stated that he had just one thing to complain about: subsystems that are not marking patches for stable. Here, Greg mentioned a few of those subsystems, and at the same time singled Dave Miller out for praise, noting that Dave was doing a lot of "heavy lifting" for networking. Greg then opened the session for feedback from others about stable kernel management. 

Ted Ts'o noted ""I'd love to be able to mark some less urgent patches as 'stable-deferred', so that if people discover regressions, I have a chance to pull them back."" Greg said that that he would try to implement this functionality, as it is a good idea. 

A few people wanted to understand more clearly the criteria that determine whether a patch should be sent for the stable series, and others noted that there seemed to be some latitude as to what Greg considered to be an acceptable patch. Greg acknowledged the latter point, with the statement that he trusted subsystem maintainers to make the call about what patches should be sent to `stable@vger.kernel.org`. As far as choosing which patches should be sent into `stable`, people were of course reminded of `[Documentation/stable_kernel_rules.txt](<http://www.kernel.org/doc/Documentation/stable_kernel_rules.txt>)` and the summary rationale for `stable`: if the patch would be of interest for distributions aiming to produce a stable kernel for a distribution release, then that patch should be submitted to `stable`. 

James Bottomley stated that he got a lot of patches for SCSI that don't apply to the stable kernel, so he strips the stable tag from them. He asked: ""what should be done in that case?"" Greg answered that he should leave that tag on, and then respond to the automated email he will get when the patch fails to apply to the stable kernel tree with the correct patch for that older kernel tree. 

Greg concluded by asking whether the current release pace of the stable series was okay. There was general agreement that the pace--a release every one to two weeks--was good, and many people expressed appreciation for the excellent job Greg is doing on the stable kernel.
