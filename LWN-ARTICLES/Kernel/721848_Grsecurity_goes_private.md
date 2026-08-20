---
title: Grsecurity goes private
url: https://lwn.net/Articles/721848/
date: "May 4, 2017"
category: "Security-Security technologies"
author: "By Jonathan Corbet May 4, 2017"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 4, 2017

On April 26, the grsecurity project [announced](<https://grsecurity.net/passing_the_baton.php>) that it was withdrawing public access to its kernel-hardening patch sets; henceforth, they will be available only to paying customers of Open Source Security, Inc., the company behind this work. This move has yielded quite a bit of discussion and no small amount of recrimination. It is not clear, though, that the right conclusions are being drawn from this change. 

The grsecurity patch set is intended to harden the kernel against a wide range of potential attacks. Its mitigations range from simple techniques like making important data structures read-only to relatively advanced defenses like [RAP](<https://grsecurity.net/rap_faq.php>). Over the years, the developers involved have certainly broken new ground and come up with some novel solutions; as a result, these patches are appreciated by a fair community of users — a community that has lost free access to this work in the future. 

#### Grsecurity and the mainline

These patches have not, as a whole, found their way into the mainline kernel, for a number of reasons. One is the simple fact that the kernel development community historically has not fully appreciated the need for kernel hardening. There was an increasingly unwarranted level of confidence in the kernel's inherent security and a lack of desire to face the trade-offs that often come with hardening patches. As a result, Linux fell behind the state of the art in this area, and external patches like grsecurity were the best way to increase the kernel's resistance to attacks. 

Over the last couple of years or so, there has been a shift in the kernel community that has made it more open to hardening work; the ongoing efforts of the grsecurity developers certainly deserve some credit for this change. As a result, a number of technologies, most inspired by the grsecurity work, have made their way into the mainline, though the code has often changed significantly (or been entirely rewritten) along the way. Current kernels have [address-space layout randomization](<https://lwn.net/Articles/569635/>), [post-init read-only memory](<https://lwn.net/Articles/666550/>), [hardened user-space access routines](<https://lwn.net/Articles/695991/>), [GCC plugins](<https://lwn.net/Articles/691102/>) for the build process, [reference-count hardening](<https://lwn.net/Articles/693038/>), [virtually mapped kernel stacks](<https://lwn.net/Articles/692953/>), and more. There have been numerous complaints and snide comments, especially from the grsecurity camp ([example](<https://forums.grsecurity.net/viewtopic.php?f=7&t=3367>)), on how these features have been implemented, and it may well be that many of them are not as strong as one would like. But they represent progress nonetheless. 

Why not just merge the grsecurity patches outright? Because that is not how the kernel community works. Changes to the kernel are split into small, reviewable changes, (sometimes) documented, and each is debated on its own merits. The grsecurity patches are not split in this manner and, in many cases, they do not meet the kernel community's standards in their current form. Like it or not, there _are_ trade-offs to be made with any patch, including security patches, and those trade-offs, which may include performance and long-term maintainability issues, must be discussed. 

Security patches tend to be discussed more fiercely than others, partly because some kernel developers see a relatively small advantage to them for a relatively large cost. The value of kernel hardening is more apparent to some than to others. But security is not unique in this way; getting invasive changes of just about any type into the core kernel is never an easy task. The grsecurity developers have had little interest in doing the work to get their patches into the mainline; the same is true of the bulk of those who use the grsecurity patch set. 

Nothing obligates any of these people to put in that effort, but the implication is that this work will fall to others who are interested in seeing these patches upstreamed. Finding developers willing to take that task on and deal with the ensuing discussions has not always been an easy task, but the pace of upstreaming has increased in recent years anyway. Much of this work has been done under the aegis of the Kernel Self-Protection Project (KSPP), an informal group of whom founder Kees Cook is the most visible member. 

#### Why grsecurity went away

The withdrawal of the public grsecurity patch set has led to a certain amount of finger pointing and recrimination, with many blaming KSPP or just about anybody except the grsecurity developers themselves for the change. Consider, for example, [this statement](<https://hardenedlinux.github.io/announcement/2017/04/29/hardenedlinux-statement2.html>) from a group that calls itself "HardenedLinux" (or "h4rdenedzer0"): 

As the result of a discussion inside h4rdenedzer0, we believe that Linux foundation is the culprit behind all this result that the commercial/individual/community users losing access to the test patches. 

Or [this post](<https://lwn.net/Articles/721852/>) from Mathias Krause: 

I think the main reason for Brad [Spengler] and PaX Team to make their work private is the increased amount of work [KSPP] has put on them without providing any valuable work in return. They just don't want to be forced to maintain and fix-up the variants of grsecurity/PaX features KSPP lands upstream. 

Both claims are somewhat difficult to back up. 

The Linux Foundation does not control kernel development and does not run KSPP. What the Linux Foundation _has_ done is to fund, via the Core Infrastructure Initiative, some of the work to bring grsecurity features into the mainline. Paying for work on free software, intended to improve the security of the Linux kernel, does not seem like a particularly blameworthy activity, so it's not entirely clear why the Linux Foundation is being faulted here. There is talk of too many press releases; these do exist, including [one issued on May 4](<https://www.coreinfrastructure.org/news/blogs/2017/05/cii-advances-kernel-security>), but it seems strange to expect that the Linux Foundation should not highlight the work it is doing. 

The claim that KSPP is somehow making life harder for grsecurity is also hard to justify. Those developers need not participate in this upstreaming work, and they need not accept the results of that work if they prefer their own patches. They _do_ have to keep up with the pace of kernel development in general if they want the work that the rest of the community has been doing, but KSPP has not created that issue — that is a well-known cost of maintaining an out-of-tree patch set. The upstreaming work has also brought benefits to grsecurity, demonstrated by the fact that grsecurity has incorporated that work into its own patch set, as [outlined in detail](<https://lwn.net/Articles/721750/>) by Cook. 

Much of the discussion seems based on the notion that the kernel community somehow owes something to the grsecurity developers for their work. That viewpoint overlooks something important: the entire reason Open Source Security Inc. is in business in the first place is that it gets an entire kernel, for free, that is actively maintained and extended by about 4,000 developers every year. It is a huge gift for a group of developers who, say, want to create a security consulting company. They can never repay the value of that gift; indeed, even the largest contributors to the kernel get more from it than they put in. That is why they are contributors in the first place. 

In a sense, all users of the kernel owe a debt to everybody who has contributed over the years. And every one of us who has contributed has made a gift to all Linux users. The value of the grsecurity patches is significant, but it pales in respect to the value of the kernel as a whole. Grsecurity is not special in relation to the vast body of other contributions on which it is built. 

So public access to grsecurity was not withdrawn due to workload, and it was not withdrawn because its developers have been somehow cheated by the development community. There is a much more likely explanation here: the movement of hardening features into the mainline kernel poses challenges to the developers' business model, which depends on providing security features not found there. Closing off access is meant to slow that movement and preserve the differentiation of the grsecurity patch set. This decision is within their right to make, as long as they stay within the terms of the GPL (and there have been no serious claims that they have done otherwise), but it should be seen as what it is: a business move intended to keep their work proprietary in spirit. 

The existing patch set will continue to exist, of course, and it may well be that some suitably motivated developers will work to keep it updated with future kernel releases. Meanwhile, the work to bring more security features into the mainline kernel will continue. The kernel will get more secure over time, even without the minimal involvement of the grsecurity developers. Hopefully they will be successful in their business and will draw some satisfaction that, through the process of upstreaming, their older work will eventually improve the security of hundreds of millions of kernel users.
